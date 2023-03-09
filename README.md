# sigstore-demo
Demo of how to use Sigstore's Cosign to sign files both locally and via Github Actions. For local testing you will need [Go](https://go.dev/doc/install), and [Cosign](https://docs.sigstore.dev/cosign/installation/) installed

## Signing tarball with local keys

1. create an ```artifact.txt``` file ```echo 'My Artifact' > artifact.txt```, and create a tarball to sign ```tar -czvf artifact.tar.gz artifact.txt```
1. Ensure that you do not commit generated keys to Github update your ```.gitignore``` file ```echo '*.key'>>.gitignore```
1. In this example, Artifact will be signed via a local key pair which you will create via cosign ```cosign generate-key-pair```
1. By default ```cosign sign``` will assume that you are signing a container image, as we're signing a local file use ```cosign sign-blob --key cosign.key -y --output-signature signature.sig  --tlog-upload=false artifact.tar.gz```. The ```-y``` will auto accept the warning about publishing information, and the ```--output-signature``` sill store the signature in a file.
1. Verify the signature with ```cosign verify-blob --key cosign.pub --signature signature.sig  --insecure-ignore-tlog artifact.tar.gz```. You should receive the output of ```Verified OK``` as the file has not been tampered with.
1. Move your original ```artifact.txt``` to a new name ```mv artifact.txt original-artifact.txt``` and create a new one with a modified value ```echo 'My Evil Artifact' > artifact.txt'``` then create a new tarball ```tar -czvf artifact.tar.gz artifact.txt```
1. Now attempt to verify the file again ```cosign verify-blob --key cosign.pub --signature signature.sig  --insecure-ignore-tlog artifact.tar.gz```. You will get an error here as the signature will no longer match as the ```artifact.tar.gz``` has been modified.  

This was a local signing example with disposable keys, and because when ```sign-blob``` was used ```--tlog-upload=false``` was provided as a command line flag. This test would not be uploaded to the Rekor signature transparency log. 


## Keyless Signing of a tarball using Fulcio GA
These commands will uploaded to Rekor and can include identifiable info, so be aware of the commands you are running. To ensure you can verify things install the [Rekor cli](https://docs.sigstore.dev/rekor/installation/). When installing via ```go install``` you might also have to set your Go path ```export GOPATH="$HOME/go"``` and update your system path ```PATH="$GOPATH/bin:$PATH"```

1. create an ```artifact.txt``` file ```echo 'My Artifact' > artifact.txt```, and create a tarball to sign ```tar -czvf artifact.tar.gz artifact.txt```
1. Ensure that you do not commit generated keys to Github update your ```.gitignore``` file ```echo '*.key'>>.gitignore```
1. By default ```cosign sign``` will assume that you are signing a container image, as we're signing a local file use ```cosign sign-blob --issue-certificate artifact.tar.gz```
1. The command will output a URL in ```https://oauth2.sigstore.dev/auth/auth?...```. You will Authenticate using via OIDC to you Github account to gain a certificate which is signing the ephemeral keys. Please note this email address/ID will be added to the public transparency log. Once you successfully login you will be provided with a code, copy and paste this back into the terminal
1. A successful signature will output a certificate signed by the Fulcio Certificate Authority, transparency log ID, and a Signature. You can verify that this has been appended to the transparency log using the Rekor CLI ```rekor-cli get --rekor_server https://rekor.sigstore.dev --log-index OUTPUT_FROM_SIGN_BLOB  --format=json | jq -r .Body```. Using the JSON output and ```jq``` to make things easier to read. Rekor will return a record of your transaction. This will be captured in a ```HashedRekordObj``` which has a ```signature``` inside the ```signature``` there will be: 
	1. ```signature``` this will contain the same signature which was output as a part of the ```cosign sign-blob``` command above. In the local key example this was captured in ```signature.sig```
	1. ```publicKey``` this will be an x.509 certificate which was issued from Sigstore's certificate authority [Fulcio](https://docs.sigstore.dev/fulcio/overview/). In order to be issued this certificate you had to authenticate via Github account. This certificate contains information about both the OICD provider, and the identity (email address) which was used. 
1. Create the following variables from Rekor information to verify the signature: 
	1. $logIndex: capture the Log Index from the output of ```cosign sign-blob```: ```logIndex=OUTPUT_FROM_SIGN_BLOB```
	1. $uuid:   ```uuid=$(rekor-cli get --rekor_server https://rekor.sigstore.dev --log-index $logIndex --format=json | jq -r .UUID)```
	1. $sig:  ```sig=$(rekor-cli get --uuid=$uuid --format=json | jq -r .Body.HashedRekordObj.signature.content)```
	1. $cert: ```cert=$(rekor-cli get --rekor_server https://rekor.sigstore.dev --log-index $logIndex  --format=json | jq -r .Body.HashedRekordObj.signature.publicKey.content)```
1. If you want to view the certificate you must decode it ```echo $cert | base64 --decode | openssl x509 -text```   
1. You can now verify the signature using ```cosign verify-blob --cert <(echo $cert | base64 --decode) --signature <(echo $sig | base64 --decode) --certificate-identity Email_Authetnicated_to_Github --certificate-oidc-issuer https://github.com/login/oauth artifact.tar.gz```. This verification process checks not just is the hashed value of the code not changed, also that the identity that requested the temporary keys was authenticated, and the key was valid at the time of signing. This entry is also a part of the immutable Rekor transparency log
1. Manipulate the value of the artifact ```echo 'My Evil Artifact' > artifact.txt'``` then create a new tarball ```tar -czvf artifact.tar.gz artifact.txt```
1. Attempt to validate the blob again, and you will receive an error, as the signature will no longer match. ```COSIGN_EXPERIMENTAL=1 cosign verify-blob --cert <(echo $cert | base64 --decode) --signature <(echo $sig | base64 --decode) --certificate-identity Email_Authetnicated_to_Github --certificate-oidc-issuer https://github.com/login/oauth artifact.tar.gz```
1. To allow a customer to validate the signature of your code you would need to publish
	1. The source code you wish to release
	1. The Signature produced by ```cosign sign-blob```
	1. The Certificate issued by Fulcio
	1. The Identity which was authenticated to request a certificate from Fulcio, and the OIDC URL 
