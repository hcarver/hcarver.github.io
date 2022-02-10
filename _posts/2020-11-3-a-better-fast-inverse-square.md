---
layout: post
title: 'A better "fast inverse square"'
date: 2020-11-3
status: published
type: post
published: true
author: Hywel Carver
---

I'm going to keep this as short as I can.

Fast Inverse Square Root (let's abbreviate that to `InvSqu`, but sometimes it's just called `0x5F3759DF` apparently) is a famous bit of unclear C code that
was used in Quake 3.

It's always frustrated me that this code is held up as an example of brilliance. As an algorithm, it's clever. As code,
it's terrible. Hard to read, hard to maintain, opaque.

That's probably partly a result of the time when it was written - C rather than C++, compiler optimisation was probably
less reliable.

I think we can do better today.

### The rules

I'm going to write a new version of `InvSqu` which produces identical assembly code.

* I'm going to let myself use C++. That's an advantage the original developer didn't have.
* I'm going to use GCC 10.2, with some optimizations turned on `-O3` probably. That's an advantage the original developer probably(?) didn't have.
* I'll only be looking at the `InvSqu` function itself. I'm not bothered about the assembly from other functions I write. I'll trust the toolchain to remove any unneeded function bodies (because they'll be inlined everywhere).
* I don't care about immaterial differences in the assembly, e.g. if the output has two independent instructions in a different order.
* I'm only going to look at x86-64 output.

I'll be doing this on https://godbolt.org/ for simplicity. Feel free to try it yourself.

### V0

Here's V0 of the code ([source](https://en.wikipedia.org/wiki/Fast_inverse_square_root#Overview_of_the_code)):

```c
float Q_rsqrt( float number )
{
	long i;
	float x2, y;
	const float threehalfs = 1.5F;

	x2 = number * 0.5F;
	y  = number;
	i  = * ( long * ) &y;                       // evil floating point bit level hacking
	i  = 0x5f3759df - ( i >> 1 );               // what the fuck?
	y  = * ( float * ) &i;
	y  = y * ( threehalfs - ( x2 * y * y ) );   // 1st iteration
//	y  = y * ( threehalfs - ( x2 * y * y ) );   // 2nd iteration, this can be removed

	return y;
}
```

Producing this:

```assembly
Q_rsqrt(float):
        movss   DWORD PTR [rsp-4], xmm0
        mulss   xmm0, DWORD PTR .LC0[rip]
        mov     rdx, QWORD PTR [rsp-4]
        mov     eax, 1597463007
        movss   xmm1, DWORD PTR .LC1[rip]
        sar     rdx
        sub     eax, edx
        movd    xmm2, eax
        mulss   xmm0, xmm2
        mulss   xmm0, xmm2
        subss   xmm1, xmm0
        mulss   xmm1, xmm2
        movaps  xmm0, xmm1
        ret
.LC0:
        .long   1056964608
.LC1:
        .long   1069547520
```

### V1

The code uses an approximation (which we'll come back to) followed by an iteration of the Newton-Raphson method.
Let's extract the Newton-Raphson bit first.

```c
float newtonRaphson(float x, float approximation) {
  const float threehalfs = 1.5F;
  float x2 = x * 0.5F;
  return approximation * ( threehalfs - ( x2 * approximation * approximation ) );
}

float Q_rsqrt( float number )
{
	long i;
	float y;

	y  = number;
	i  = * ( long * ) &y;                       // evil floating point bit level hacking
	i  = 0x5f3759df - ( i >> 1 );               // what the fuck?
	y  = * ( float * ) &i;
	y  = newtonRaphson(number, y);   // 1st iteration
//	y  = newtonRaphson(number, y);   // 2nd iteration, this can be removed

	return y;
}
```

Identical assembly so far.

### V2

Let's unpack that Newton-Raphson function a bit. The core idea is that if `x` is an approximation of the root of an equation `f`, then `x - f(x) / f'(x)` will be a better approximation.

What's clever about this implementation is it avoids division altogether. When you work through the maths, you end up with a load of multiplications instead. I'm going to add comments to make the maths easier to understand for another developer.

```c
float invSqRtNewtonRaphson(float target, float approximation) {
  // Complicated maths approaching!
  // The Newton-Raphson method is a way of improving an approximation to finding the solution of an equation
  // equal to 0.
  //
  // In the case of an inverse square root, the equation we need to solve is f(x) = x^-2 - target = 0
  // The first derivative of f(x) is f'(x) = -2 * x ^ -3
  //
  // The method says that to take a current approximation and find an improved approximation, you compute
  // approximation - f(approximation) / f'(approximation)
  //
  // For inverse square root, that is (with some simplifying):
  // approximation - (approximation ^ -2 - target) / (-2 * approximation ^ -3)
  // approximation - approximation ^ 3 * (approximation ^ -2 - target) / (-2)
  // approximation + approximation ^ 3 * (approximation ^ -2 - target) / (2)
  // approximation + (approximation - approximation ^ 3 * target) / 2
  // approximation + approximation / 2 - approximation ^ 3 * target / 2
  // approximation * (1 + 1 / 2 - approximation ^ 2 * target / 2)
  // approximation * (3 / 2 - (target / 2) * approximation ^ 2)
  // approximation * (3 / 2 - (target / 2) * approximation * approximation)
  // approximation * (3 / 2 - (target * (1 / 2) * approximation * approximation)

  return approximation * ( 1.5F - ( target * 0.5F * approximation * approximation ) );
}

float Q_rsqrt( float number )
{
	long i;
	float y;

	y  = number;
	i  = * ( long * ) &y;                       // evil floating point bit level hacking
	i  = 0x5f3759df - ( i >> 1 );               // what the fuck?
	y  = * ( float * ) &i;
	y  = invSqRtNewtonRaphson(number, y);   // 1st iteration
//	y  = invSqRtNewtonRaphson(number, y);   // 2nd iteration, this can be removed

	return y;
}
```

Same assembly output so far.

### V3

Now for the first half. That uses a linear approximation to the inverse square function, which the second half improves on. I'm also going to remove some unnecessary comments.

```c
float invSqRtNewtonRaphson(float target, float approximation) {
  // Complicated maths approaching!
  // The Newton-Raphson method is a way of improving an approximation to finding the solution of an equation
  // equal to 0.
  //
  // In the case of an inverse square root, the equation we need to solve is f(x) = x^-2 - target = 0
  // The first derivative of f(x) is f'(x) = -2 * x ^ -3
  //
  // The method says that to take a current approximation and find an improved approximation, you compute
  // approximation - f(approximation) / f'(approximation)
  //
  // For inverse square root, that is (with some simplifying):
  // approximation - (approximation ^ -2 - target) / (-2 * approximation ^ -3)
  // approximation - approximation ^ 3 * (approximation ^ -2 - target) / (-2)
  // approximation + approximation ^ 3 * (approximation ^ -2 - target) / (2)
  // approximation + (approximation - approximation ^ 3 * target) / 2
  // approximation + approximation / 2 - approximation ^ 3 * target / 2
  // approximation * (1 + 1 / 2 - approximation ^ 2 * target / 2)
  // approximation * (3 / 2 - (target / 2) * approximation ^ 2)
  // approximation * (3 / 2 - (target / 2) * approximation * approximation)
  // approximation * (3 / 2 - (target * (1 / 2) * approximation * approximation)

  return approximation * ( 1.5F - ( target * 0.5F * approximation * approximation ) );
}

float linearApproxInvSqRt ( float number ) {
	float y  = number;
	long i  = * ( long * ) &y;                  // evil floating point bit level hacking
	i  = 0x5f3759df - ( i >> 1 );               // what the fuck?
	return * ( float * ) &i;
}

float Q_rsqrt( float number ) {
  // Get a linear approximation to the solution
  float approxSolution = linearApproxInvSqRt(number);
  // Improve on the approximateSolution
	approxSolution  = invSqRtNewtonRaphson(number, approxSolution);

	return approxSolution;
}
```

Great, this is still producing identical assembly.

### V4

Now to tidy up the linear approximation function.

```c
float invSqRtNewtonRaphson(float target, float approximation) {
  // Complicated maths approaching!
  // The Newton-Raphson method is a way of improving an approximation to finding the solution of an equation
  // equal to 0.
  //
  // In the case of an inverse square root, the equation we need to solve is f(x) = x^-2 - target = 0
  // The first derivative of f(x) is f'(x) = -2 * x ^ -3
  //
  // The method says that to take a current approximation and find an improved approximation, you compute
  // approximation - f(approximation) / f'(approximation)
  //
  // For inverse square root, that is (with some simplifying):
  // approximation - (approximation ^ -2 - target) / (-2 * approximation ^ -3)
  // approximation - approximation ^ 3 * (approximation ^ -2 - target) / (-2)
  // approximation + approximation ^ 3 * (approximation ^ -2 - target) / (2)
  // approximation + (approximation - approximation ^ 3 * target) / 2
  // approximation + approximation / 2 - approximation ^ 3 * target / 2
  // approximation * (1 + 1 / 2 - approximation ^ 2 * target / 2)
  // approximation * (3 / 2 - (target / 2) * approximation ^ 2)
  // approximation * (3 / 2 - (target / 2) * approximation * approximation)
  // approximation * (3 / 2 - (target * (1 / 2) * approximation * approximation)

  return approximation * ( 1.5F - ( target * 0.5F * approximation * approximation ) );
}

float linearApproxInvSqRt ( float number ) {
	float y  = number;
	long i  = * ( long * ) &y;                  // evil floating point bit level hacking
	i  = 0x5f3759df - ( i >> 1 );               // what the fuck?
	return * ( float * ) &i;
}

float Q_rsqrt( float number ) {
  // Get a linear approximation to the solution
  float approxSolution = linearApproxInvSqRt(number);
  // Improve on the approximateSolution
	approxSolution  = invSqRtNewtonRaphson(number, approxSolution);

	return approxSolution;
}
```
