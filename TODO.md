
## Contract
- [X] Recovery
- [X] Securer -> External Entity multisig (special multisig that reference either an address, or several addresses, or a contract for the signers this allows the entity to quickly be able to change the signers in case of the address being compromise without having all the user change it. This contract needs multisginatures itself with very secure handling)
- [ ] Deep Testing (Still issue manually creating the AuthEntries linked to the invocation Tree)

## App
- [ ] Account creation
- [ ] Allowing to access the passkey if phone have access to it
- [ ] Allows creating an account on keystore otherwise
- [ ] Allow access to enclave on android
- [ ] Allow signatures
- [ ] Allow pre-signing with the app sending the 0 tx in the background

## Backend
- [ ] Integrate a backend for multisg
- [ ] Creating a service to send pre-signed tx from server



