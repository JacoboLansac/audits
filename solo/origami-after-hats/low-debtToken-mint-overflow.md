# Potential overflow of `totalPrincipal` in `OrigamiDebtToken::mint()`

The `mint()` and `_mintToDebtor()` lack overflow checks when increasing `totalPrincipal` and `toDebtor.principal`. 
This would generally not be an issue, as the function is protected with the `onlyMinters` modifier, but still worth including a requirement. 

```javascript
    /**
     * @notice Approved minters can add a new debt position on behalf of a user.
     * @param _debtor The address of the debtor who is issued new debt
     * @param _mintAmount The notional amount of debt tokens to issue.
     */
    function mint(address _debtor, uint256 _mintAmount) external override onlyMinters {
        if (_debtor == address(0)) revert CommonEventsAndErrors.InvalidAddress(_debtor);
        if (_mintAmount == 0) revert CommonEventsAndErrors.ExpectedNonZero();

        emit Transfer(address(0), _debtor, _mintAmount);
@>      _mintToDebtor(_debtor, _mintAmount.encodeUInt128());
    }

```

```javascript
    function _mintToDebtor(address _debtor, uint128 _amount) internal {
        Debtor storage toDebtor = debtors[_debtor];
        DebtorPosition memory _debtorPosition = _getDebtorPosition(toDebtor);  // updates storage, and returns memory debtorPosition

@>      // @audit-issue LOW: no control on overflows on totalPrincipal or toDebtor.principal

        unchecked {
@>          totalPrincipal += _amount;
@>          toDebtor.principal = _debtorPosition.principal = _debtorPosition.principal + _amount;  // both memory and storage updated
        }

        emit DebtorBalance(_debtor, _debtorPosition.principal, _debtorPosition.interest);
    }

```

**Impact: low**

Likelihood: Low, as it would require either a compromised wallet with the `minter` role or a mistake. 
Impact: high, as it would mess up completely the accounting of the `principal` debt. 

**Recommendation**

Move the increment on `totalPrincipal` outside the unchecked box to let the built-in solidity overflow checker do its job. 
If the `totalPrincipal` doesn't overflow, the `toDebtor.principal` cannot overflow, as the `totalPrincipal` is the sum of all individual debtor's principals. 

```diff
    function _mintToDebtor(address _debtor, uint128 _amount) internal {
        Debtor storage toDebtor = debtors[_debtor];
        DebtorPosition memory _debtorPosition = _getDebtorPosition(toDebtor);  // updates storage, and returns memory debtorPosition

+       totalPrincipal += _amount;

        unchecked {
-           totalPrincipal += _amount;
            toDebtor.principal = _debtorPosition.principal = _debtorPosition.principal + _amount;
        }

        emit DebtorBalance(_debtor, _debtorPosition.principal, _debtorPosition.interest);
    }

```
