Just some thoughts. In AbstractLovTokenManager::maxExit(), and morpho::_maxRedeemFromReserves() . The maxExit() docstring says it checks:
- max reserves available until hitting the A/L ratio floor
- Any other constraints from the underlying implementation.

This would look to me like "the smaller of the two above, is the limiting factor, and therefore it is what maxExit() should output. 
However, this is not the case, as the second point reverts as soon as the A/L floor falls outside the healthy range of the borrow manager. 

Imagine this scenario:
According to ALratio flor, the maxExit  is 1000 lovTokens.
According to the underlying, (i.e. morphos the max exit before hitting the unhealthy LLTV), would be 900 lovTokens.

With the current implementation, maxExit()  does not return 900 lovTokens, but returns 0, because _maxRedeemFromReserves() returns 0 as soon as _borrowLend.isSafeAlRatio returns false
Image
Image
I think basically what I'm pointing out is that morpho::_maxRedeemFromReserves() is supposed to return the "max" redeemable, but it returns zero as soon as the userALrange.floor falls into unhealthy LLTV
Perhaps it is a concious decission, but it doesn't match what the docstring of _maxRedeemFromReserves() suggests


