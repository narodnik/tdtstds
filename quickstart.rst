.. _tut-quickstart:

***************
Quickstart
***************

We will go more in depth into these concepts during later chapters.

::

    use darkfutures::*;
    fn main() {
        //
        // Initialization
        //

        let number_attributes = 2;
        let threshold_service = 5;
        let total_services = 5;

        let (secret_keys, verify_key) =
            generate_keys(number_attributes, threshold_service, total_services);
        let coconut =
            Coconut::<OsRngInstance>::new(number_attributes, threshold_service, total_services);

        // Create our services that will sign new credentials
        let mut services: Vec<_> = secret_keys
            .into_iter()
            .enumerate()
            .map(|(index, secret)| {
                Service::from_secret(&coconut, secret, verify_key.clone(), (index + 1) as u64)
            })
            .collect();

        //
        // Deposit
        //

        // wallet: Deposit a new token worth 110 credits
        let token_value = 110;
        let token_secret = TokenSecret::generate(token_value, &coconut);

        println!("Deposit started...");

        let token = {
            // wallet: Create a new transaction
            let mut tx = Transaction::new();
            // wallet: Create a single output
            let (output, mut output_secret) = Output::new(&coconut, &token_secret);

            // wallet: We are depositing 110, so expect a new token of 110 to be minted
            tx.add_deposit(token_secret.value);
            let output_id = tx.add_output(output);

            // Once we have added the inputs and outputs, we must call this function...
            let (input_blinds, output_blinds) =
                tx.compute_pedersens(&coconut, &vec![], &vec![token_value]);

            // wallet: Then for every input/output we created, call this one.
            output_secret.setup(output_blinds[output_id]);
            // wallet: Now start to generate the proofs
            let output_proof_commits = output_secret.proof_commits();

            let mut hasher = HasherToScalar::new();
            output_proof_commits.commit(&mut hasher);
            let challenge = hasher.finish();

            // Rust is gay
            std::mem::drop(output_proof_commits);

            // wallet: Finish proof and add it to our transaction
            let output_proofs = output_secret.finish(&challenge);
            tx.outputs[output_id].set_proof(output_proofs);

            // Also set the challenge computed from all the proofs in our transaction
            tx.challenge = challenge;

            // service: Each service will now validate and sign the transaction
            let output_signatures: Vec<_> = services
                .iter_mut()
                .enumerate()
                .map(|(index, service)| match service.process(&tx) {
                    Ok(signatures) => {
                        println!("Service-{} signed {} tokens", index, signatures.len());
                        signatures
                    }
                    Err(err) => {
                        panic!("Error occured signing (service={}): {}", index, err);
                    }
                })
                .collect();

            // wallet: Unblind and accept the returned signed token if signed by at least
            //         M of N services.
            let mut tokens = tx.unblind(&coconut, &vec![&token_secret], output_signatures);
            assert!(tokens.len() == 1);
            tokens.pop().unwrap()
        };

        println!("Deposit finished.");
    }
