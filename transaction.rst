**********************
Creating a Transaction
**********************

Theory
======

Transactions have 4 components:

* **Deposits**. When we create new tokens, this plaintext field must be set to the total value of new tokens being created. We must also crate the relevant outputs.
* **Withdraws**. This field is concerned with tokens being burned. The tokens burned are set in the input.
* **Inputs**. Tokens going into the transaction that will be destroyed.
* **Outputs**. New tokens that will be minted.

Importantly, the money going in must equal the money going out:

.. math::

   \sum{\operatorname{inputs}} + \sum{\operatorname{deposits}}
   =
   \sum{\operatorname{outputs}} + \sum{\operatorname{withdraws}}

This is ensured through zero-knowledge proofs.

Examples
--------

Practice: Depositing a Token
============================

First create the token secret.

::

    let token_value = 110;
    let token_secret = TokenSecret::generate(token_value, &coconut);

Create a new transaction.

::

    let mut tx = Transaction::new();

Create a single output.

::

    let (output, mut output_secret) = Output::new(&coconut, &token_secret);

We are depositing 110, so expect a new token of 110 to be minted.

::

    tx.add_deposit(token_secret.value);
    let output_id = tx.add_output(output);

Practice: Splitting a Token
===========================

Construct a new transaction.

::

    let mut tx = Transaction::new();

Create the inputs and outputs.

::

    let (input, mut input_secret) = Input::new(&coconut, &verify_key, &token, &token_secret);
    let (output1, mut output1_secret) = Output::new(&coconut, &token1_secret);
    let (output2, mut output2_secret) = Output::new(&coconut, &token2_secret);

Add them to the transaction.

::

    let input_id = tx.add_input(input);
    let output1_id = tx.add_output(output1);
    let output2_id = tx.add_output(output2);


