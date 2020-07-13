# snarkjs: JavaScript implementation of zkSNARKs.

This is a JavaScript and Pure Web Assembly implementation of zkSNARK schemes. It uses the Groth16 Protocol (3 point only and 3 pairings)

This library includes all the tools for the Trusted setup multiparty ceremony.
This includes the universal ceremony "powers of tau".
And the per circuit phase 2 ceremony.

The formats used in this library for the multipary computation are compatible with the ones used in other (implementations in rust)[].

This library uses the compiled circuits generated by the circom compiler.

The library works in nodejs and browser.

It's a ESM module, so it can be directly imported from bigger projects using rollup or webpack.

The low level criptography is done directly in wasm. And it uses working threads to parallelize the computations. The result is a high performance library with benchmarks comparable with implementations running in the host.

## Usage / Tutorial.

### Install snarkjs and circom
```sh
npm install -g circom@latest
npm install -g snarkjs@latest
```

### Help

```sh
snarkjs --help
```

In commands that takes long time, you can add the -v or --verbose option to see the progress.

The help for specific command:

Example
```sh
snarkjs groth16 prove --help
```

Most of the commands have a shor alias.

For example, the previos command can also be invoked as:

```sh
snarkjs g16p --help
```

For this tutorial, create a new direcory and change the path
```sh
mkdir snarkjs_example
cd snarkjs_example
```


### Start a new ceremony.

```sh
snarkjs powersoftau new bn128 12 pot12_0000.ptau
```

You can also use bls12-381 as the curve.

The secons parameter is the power of two of the maximum number of contraints that can accept this ceremony.

In this case 12 means that the maximum constraints will be 2**12 = 4096

### Contribute in the ceremony
```sh
snarkjs powersoftau contribute pot12_0000.ptau pot12_0001.ptau --name="Example Name" -v
```

The name is a random name and it's include for reference. It's printed in the verification.

### Do a second contribution
```sh
snarkjs powersoftau contribute pot12_0001.ptau pot12_0002.ptau --name="Second contribution Name" -v -e="some random text"
```

the -e parameter allows the comman to be non interactive and use this text as an extra source of entropy for the random generation.


### Verify the file
```sh
snarkjs powersoftau verify pot12_0002.ptau
```

This command checks all the contributions of the Multiparty Computation (MPC) and list the hashes of the
intermediary results.

### Contribute using ther party software.

```sh
snarkjs powersoftau export challange pot12_0002.ptau challange_0003
snarkjs powersoftau challange contribute bn128 challange_0003 response_0003
snarkjs powersoftau import response pot12_0002.ptau response_0003 pot12_0003.ptau -n="Third contribution name"
```


### Add a beacon
```sh
snarkjs powersoftau beacon pot12_0003.ptau pot12_beacon.ptau 0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f 10 -n="Final Beacon"
```

### Prepare phase2
```sh
snarkjs powersoftau prepare phase2 pot12_beacon.ptau pot12_final.ptau -v
```

### Verify the last file
```sh
snarkjs powersoftau verify pot12_final.ptau
```

### Create a circuit
```sh
cat <<EOT > circuit.circom
template Multiplier(n) {
    signal private input a;
    signal private input b;
    signal output c;

    signal int[n];

    int[0] <== a*a + b;
    for (var i=1; i<n; i++) {
    int[i] <== int[i-1]*int[i-1] + b;
    }

    c <== int[n-1];
}

component main = Multiplier(1000);
EOT
```

This is an example circom fille that allows to test the system with different number of contraints.

In this case 1000, but it can be changed to any nomber of constraints.

### compile the circuit
```sh
circom circuit.circom -r -w -s -v
```

-r to generate the .r1cs file
-w to generate the .wasm file that computes the witness from an input.
-s to generate the .sym file that contains the human readable names of all signals. (Important to debug the circuit)
-v Verbose. To see the progress of the compilation.

### info of a circuit
```sh
snarkjs r1cs info circuit.r1cs
```

### Print the constraints
```sh
snarkjs r1cs print circuit.r1cs
```

### export r1cs to json
```sh
snarkjs r1cs export json circuit.r1cs circuit.r1cs.json
cat circuit.r1cs.json
```


### Generate the reference zKey without contributions from the circuit.
```sh
snarkjs zkey new circuit.r1cs pot12_final.ptau circuit_0000.zkey
```

circuit_0000.zkey does not include any contribution yet, so it cannot be used in a final circuit.

### Contribute in the phase2 ceremony
```sh
snarkjs zkey contribute circuit_0000.zkey circuit_0001.zkey --name="1st Contributor Name" -v
```

### Do a second phase2 contribution
```sh
snarkjs zkey contribute circuit_0001.zkey circuit_0002.zkey --name="Second contribution Name" -v -e="Another random entropy"
```


### Verify the zkey file
```sh
snarkjs zkey verify circuit.r1cs pot12_final.ptau circuit_0002.zkey
```


### Contribute using third party software.

```sh
snarkjs zkey export bellman circuit_0002.zkey  challange_phase2_0003
snarkjs zkey bellman contribute bn128 challange_phase2_0003 response_phase2_0003
snarkjs zkey import bellman circuit_0002.zkey response_phase2_0003 circuit_0003.zkey -n="Third contribution name"
```


### Add a beacon
```sh
snarkjs zkey beacon circuit_0003.zkey circuit_final.zkey 0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f 10 -n="Final Beacon phase2"
```

### Verify the final file
```sh
snarkjs zkey verify circuit.r1cs pot12_final.ptau circuit_final.zkey
```

### Export the verification key
```sh
snarkjs zkey export verificationkey circuit_final.zkey verification_key.json
```

### Calculat witess
```sh
cat <<EOT > input.json
{"a": 3, "b": 11}
EOT
snarkjs wtns calculate circuit.wasm input.json witness.wtns
```


### Debug witness calculation

In general, when you are developing a new circuit you will want to check for some errors in the witness calculation process.

You can do it by doing
```sh
snarkjs wtns debug circuit.wasm input.json witness.wtns circuit.sym --trigger --get --set
```

This will log every time a new component is started/ended ( --trigger ) when a signal is set (--set) and when it's get (--get)


### Proof calculation
```sh
snarkjs groth16 prove circuit_final.zkey witness.wtns proof.json public.json
```

It is possible also to do the calculate witness and the prove calculation in the same command:
```sh
snarkjs groth16 fullprove input.json circuit.wasm circuit_final.zkey proof.json public.json
```


### Verify
```sh
snarkjs groth16 verify verification_key.json public.json proof.json
```

### Export Solidity Verifier
```sh
snarkjs zkey export solidityverifier circuit_final.zkey verifier.sol
```

You can deploy th "Verifier" smartcontract using remix for example.

In order to simulate a verification call, you can do:

```sh
zkey export soliditycalldata public.json proof.json
```

And cut and paste the resolt directlly in the "verifyProof" field in the deployed smart contract.

This call will return true if the proof and the public data is valid.


## Use in node

```sh
npm install snarkjs
```

```js
const snarkjs = require("snarkjs");
const fs = require("fs");

async function run() {
    const { proof, publicSignals } = await snarkjs.groth16.fullProve({a: 10, b: 21}, "circuit.wasm", "circuit_final.zkey");

    console.log("Proof: ");
    console.log(JSON.stringify(proof, null, 1));

    const vKey = JSON.parse(fs.readFileSync("verification_key.json"));

    const res = await snarkjs.groth16.verify(vKey, publicSignals, proof);

    if (res === true) {
        console.log("Verification OK");
    } else {
        console.log("Invalid proof");
    }

}

run().then(() => {
    process.exit(0);
});
```

## Use in the web

load snarkjs.min.js and start using it normally.


```html
<!doctype html>
<html>
<head>
  <title>Snarkjs client example</title>
</head>
<body>

  <h1>Snarkjs client example</h1>
  <button id="bGenProof"> Create proof </button>

  <!-- JS-generated output will be added here. -->
  <pre class="proof"> Proof: <code id="proof"></code></pre>

  <pre class="proof"> Result: <code id="result"></code></pre>


  <script src="snarkjs.min.js">   </script>


  <!-- This is the bundle generated by rollup.js -->
  <script>

const proofCompnent = document.getElementById('proof');
const resultComponent = document.getElementById('result');
const bGenProof = document.getElementById("bGenProof");

bGenProof.addEventListener("click", calculateProof);

async function calculateProof() {

    const { proof, publicSignals } =
      await snarkjs.groth16.fullProve( { a: 3, b: 11}, "circuit.wasm", "circuit_final.zkey");

    proofCompnent.innerHTML = JSON.stringify(proof, null, 1);


    const vkey = await fetch("verification_key.json").then( function(res) {
        return res.json();
    });

    const res = await snarkjs.groth16.verify(vkey, publicSignals, proof);

    resultComponent.innerHTML = res;
}

  </script>

</body>
</html>
```


## License

snarkjs is part of the iden3 project copyright 2018 0KIMS association and published with GPL-3 license. Please check the COPYING file for more details.
