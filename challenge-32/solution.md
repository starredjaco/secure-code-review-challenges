# How to Hack

The app in this challenge is vulnerable to a Business Logic flaw in the coupon system. The `/apply-coupon` endpoint lets you stack the same coupon as many times as you want, until the cart total drops to zero (or even below, which gets clamped to 0).

The bug is in this line:

```js
cart.discountCents += Math.floor(subtotal(cart) * coupon.percentOff / 100);
```

Two things go wrong here:

1. The discount is **added** (`+=`) to whatever is already on the cart, instead of being set based on the current state.
2. There's no check whether the coupon has already been applied to this cart.

So the exploit is simply: create a cart, add an item, and then hit `/apply-coupon` over and over. Each call tacks on another 10% of the subtotal. After 10 applications, the discount equals the subtotal and the total is 0.

# How to Fix

The fix is to treat coupons as a property of the cart, not as a running tally on the discount. A clean version looks like this:

1. Track the applied coupon on the cart itself (e.g. `cart.couponCode`), and reject the request if a coupon is already applied.
2. Compute the discount fresh from the subtotal whenever it's needed, instead of mutating `discountCents` on every call.

That way it doesn't matter how many times someone hits the endpoint — the discount is always `subtotal * percentOff / 100`, and a second coupon attempt gets a `400`.

On top of that, business logic like this should always have server-side limits as defense in depth: a maximum discount percentage per cart, a per-coupon usage counter, and ideally a rule that a discount can never exceed the subtotal in the first place. Coupon abuse is one of those bugs that rarely shows up in a SAST scan, so the only real defense is making the rules explicit in code.
