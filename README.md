# Curve Fitting Approach — Explanation

## Problem, in short

We're given a set of (x, y) points that lie on a parametric curve with three unknowns — theta, M, and X — and an unknown parameter t (in the range 6 to 60) for every point. The task is to recover theta, M, and X.

The tricky part is that t isn't given to us. Each point corresponds to some t, but we don't know which one. So this isn't a plain curve-fit — we have to estimate 1500 unknown t-values along with the 3 real unknowns.

## Approach: alternate between the two unknowns

The equations are linear-ish in theta and X, but t enters through cos, sin, and an oscillating `exp(M|t|)*sin(0.3t)` term, so there's no clean closed-form way to solve for everything at once. The approach used here is a standard trick for this kind of separable problem — alternate between the two halves until both settle:

- **Step 1:** Guess theta, M, X. Guess a spread of t-values (just evenly spaced across 6–60 to start).
- **Step 2:** With theta, M, X fixed, find the best t for each point separately — the t that puts the curve closest to that point. Since `sin(0.3t)` oscillates, a plain gradient search can get stuck, so a coarse grid search over t is run first to land in the right neighborhood, then a bounded local search refines it.
- **Step 3:** With the t-values now fixed, fit theta, M, X to all 1500 points at once using least-squares (scipy's `least_squares`, with bounds matching the problem's given ranges).
- **Step 4:** Repeat steps 2 and 3. Each round the t-values and the parameters both get a little more accurate. Stop once the t-values barely move between rounds.

In practice, this settles down fast — within about 15–20 rounds the biggest change in any t-value drops below 0.0001, and theta/M/X stop moving at all.

## Checking the result

Once it converges, the fitted curve is compared back against the original points: for each point, we take `|predicted_x - actual_x| + |predicted_y - actual_y|` and average it. That number came out to about 0.000004 per point — essentially zero, which is a good sign that the recovered values (theta = 30 deg, M = 0.03, X = 55) aren't just a decent fit but the actual values used to generate the data. Real fits to noisy data don't usually land on round numbers this cleanly by accident.

Two plots back this up: one overlays the fitted curve on the scatter of given points (they sit right on top of each other), and one shows the per-point L1 error, which stays flat and tiny across the whole range of t rather than drifting or spiking anywhere.

## A note on the scoring criteria (my own reading of it)

My honest read is that the L1 number only really means something once it's checked against the true curve at matched t-values, not just against the given sample points — a method could overfit those 1500 points and still miss the underlying curve elsewhere. I don't have the true curve to check against, so the residual reported here is the best available proxy: how well the fitted equation reproduces the exact data it was given. It happens to be near-zero, which is reassuring, but worth flagging as a limitation rather than claiming more certainty than is warranted.