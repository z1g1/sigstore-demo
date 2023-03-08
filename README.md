# sigstore-demo
repository for a demo of using sigstore to sign a file with github actions


## Signing tarball with local keys
*doing this on Ubuntu*

1. If you do not have Go installed, [do that](https://go.dev/doc/install)
1. Install [Cosign](https://docs.sigstore.dev/cosign/installation/), this will be used at the CLI to do the signing. 
1. create an ```artifact.txt``` file ```echo 'My Artifact' > artifact.txt```, and create a tarball to sign ```tar -czvf artifact.tar.gz artifact.txt```
1. Ensure that you do not commit generated keys to Github update your ```.gitignore``` file ```echo '*.key'>>.gitignore```
1. In this example, Artifact will be signed via a local key pair which you will create via cosign ```cosign generate-key-pair```
1. By default ```cosign``` will assume that you are signing a container image, as we're signing a local file use ```cosign sign-blob --key cosign.key -y --output-signature signanture.sig  --tlog-upload=false artifact.tar.gz```. The ```-y``` will auto accept the warning about publishing information, and the ```--output-signature``` sill store the signature in a file.
1. Verify the signature with ```cosign verify-blob --key cosign.pub --signature signature.sig  --insecure-ignore-tlog artifact.tar.gz```. You should receive the output of ```Verified OK``` as the file has not been tampered with.
1. Move your original ```artifact.txt``` to a new name ```mv artifact.txt original-artifact.txt``` and create a new one with a modified value ```echo 'My Evil Artifact' > artifact.txt'``` then create a new tarball ```tar -czvf artifact.tar.gz atifact.txt``
1. Now attempt to verify the file again ```cosign verify-blob --key cosign.pub --signature signature.sig artifact.tar.gz```. You will see an error here as the signature will no longer match as the artifact has been modified.  





cosign sign-blob --key cosign.key -y --output-signature signanture.sig --tlog-upload=false foo.txt


cosign verify-blob --key cosign.pub --signature signanture.sig --insecure-ignore-tlog foo.txt

