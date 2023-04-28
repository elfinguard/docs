===========================
BCH Payment Judger
===========================
An online judger for BCH stochastic payments
-------------------------------------------

PaymentJudger is a third-party enclave to support stochastic payment on Bitcoin Cash mainchain. It serves the inactive payees which cannot check on-chain status and broadcast transactions. Maybe the payee is a smart contract which cannot be active, or maybe it is a real person who cannot get online, such as an author who uses ElfinHost Access-Control protocol to publish videos.

The payer and PaymentJudger follow these steps:

1. The payer sign a Bitcoin Cash transaction that pays the payee and has an OP\_RETURN output contains the possibility that this transaction would be broadcasted.

2. The enclave receives this transaction, verifies it using `testmempoolaccept`, and sends the payer a signature endorsing the payer, the payee, the amount, the possibility, etc.

3. The enclave generate a VRF output based on the transaction's hashid and decide whether to broadcast this tx based on the specified possibility. The VRF output and the proof are also sent to the payer, no matter whether the tx is broadcasted or not.

The payer-signed Bitcoin Cash transaction must be a "derivable" transaction from which the `UTXO adaptor <https://github.com/elfinguard/utxoadapter>`_ can derive an EVM transaction containing one EVM log. Further more, the first OP\_RETURN output's must have at least seven pushed data elements and the seventh must be a two-byte integer indicating the probability of the payment.

PaymentJudger will serialize and sign the derived EVM log in the same way as the authorizer does: serialize the following solidity struct using `abi.encodePacked` and sign it using `personal_sign`.

.. code-block:: solidity

    struct LogInfo {
        uint chainId; // which chain did this log happen on?
        uint timestamp; // when did this log happend at?
        address sourceContract; // which contract generates this log?
        bytes32[] topics; // a log has 0~4 topics
        bytes data; // a log has variable-length data
    }


According to the UTXO adaptor's specification, the `data` field of `LogInfo` can be decoded in following way:

.. code-block:: solidity

		(
			uint256 _confirmations,
			uint256[] memory outputs,
			uint256[] memory inputs,
			bytes[] memory otherData
		) = abi.decode(logInfo.data, (uint256, uint256[], uint256[], bytes[]));


The possibility of payment is encoded in `otherData[0]`, which is the seventh data element of the first OP\_RETURN output.

PaymentJudger will return the following information to payee:

1. Prob16, the possibility of payment. It's in the range of [0, 65535]

2. Rand16: a peusdo random number from beta[30:]. It's in the range of [0, 65535]

3. VrfAlpha: the hashid of the transaction,

4. VrfBeta:  the VRF output,

5. VrfPi:    the VRF proof,

6. LogInfo:  serialized struct LogInfo,

7. LogSig:   a signature endorsing LogInfo,

If Rand16 is less than Prob16, this transaction will be broadcasted by PaymentJudger.
