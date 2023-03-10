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
1. By default ```cosign sign``` will assume that you are signing a container image, as we're signing a local file use ```cosign sign-blob -y --bundle verification.json artifact.tar.gz```
1. The command will output a URL in ```https://oauth2.sigstore.dev/auth/auth?...```. You will Authenticate using via OIDC to you Github account to gain a certificate which is signing the ephemeral keys. Please note this email address/ID will be added to the public transparency log. Once you successfully login you will be provided with a code, copy and paste this back into the terminal
1. ```verification.json``` will contain the output required to later validate the signature. 
```
	{
	  "base64Signature": "....",
	  "cert": "...",
	  "rekorBundle": {
		"SignedEntryTimestamp": "...",
		"Payload": {
		  "body": "...",
		  "integratedTime": ...,
		  "logIndex": ...,
		  "logID": "..."
		}
	  }
	}
	```
1. Using ```verification.json``` verify the signature of the file, and the authentication of the identity which requested the signature ```cosign verify-blob --bundle verification.json --certificate-oidc-issuer https://github.com/login/oauth --certificate-identity Email_Authetnicated_to_Github artifact.tar.gz```
1. If you do not know this email address you can view it on the certificate. You can extract this from ```verification.json``` using ```jq``` and decode it ```cat verification.json | jq -r .cert | base64 --decode | openssl x509 -text```
1. Manipulate the value of the artifact ```echo 'My Evil Artifact' > artifact.txt'``` then create a new tarball ```tar -czvf artifact.tar.gz artifact.txt```
1. Attempt to validate the blob again, and you will receive an error, as the signature will no longer match. ```cosign verify-blob --bundle verification.json --certificate-oidc-issuer https://github.com/login/oauth --certificate-identity Email_Authetnicated_to_Github artifact.tar.gz```
1. To allow a customer to validate the signature of your code you would need to publish
	1. The source code you wish to release
	1. The Signature stored as ```base64Signature``` in ```verification.json```
	1. The Certificate issued by Fulcio stored as ```cert``` in ```verification.json```
	1. The Email Address which was authenticated to request a certificate from Fulcio and is stored in the Certificate
	1. The OIDC URL ```https://github.com/login/oauth```


## Keyless signing via Github Actions
When developing a Github Action you can make use of [act](https://github.com/nektos/act) to avoid having to push each new version of the file to Github so you can run it via the CIL.  This relies on [Docker Engine](https://docs.docker.com/engine/install/) to run it's containers. If running on WSL make sure you follow the [extra instructions](https://docs.docker.com/desktop/windows/wsl/) to enable Docker Desktop and WSL to talk together. When using Act the first time you will be asked to select the type of container you want to run on. There's a long running [Github issues thread](https://github.com/nektos/act/issues/107) about what the image should contain vs how large an image Act should pull down to replicate the Github Actions runner. I used a ```Medium``` size to test my local Sigstore setup.

Since you're going to be using Sigstore for keyless signing it will need access to your Github ID Token. If you are using Act for local testing you will have to supply a Personal Access Token for it's local use. When you generate the token it does not need to have any permsissions as it's just being used for Authentication purposes 