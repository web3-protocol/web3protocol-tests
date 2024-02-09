# Web3:// protocol tests

The ``web3://`` protocol is made of several ERCs :

- [ERC-6860](https://eips.ethereum.org/EIPS/eip-6860) : the base web3:// protocol with auto and manual mode, basic ENS support. This updates [ERC-4804](https://eips.ethereum.org/EIPS/eip-4804) with clarifications, small fixes and changes.
- [ERC-6821](https://eips.ethereum.org/EIPS/eip-6821) (draft) : ENS resolution : support for the ``contentcontract`` TXT field to point to a contract in another chain
- [ERC-6944](https://eips.ethereum.org/EIPS/eip-6944) (draft) / [ERC-5219](https://eips.ethereum.org/EIPS/eip-5219) : New mode offloading some parsing processing on the browser side
- [ERC-7087](https://github.com/ethereum/ERCs/pull/98) (pending) : Auto mode : Add MIME type support
- [ERC-7617](https://github.com/ethereum/ERCs/pull/245) (pending): Add chunk support in ERC-6944 resource request mode
- [ERC-7618](https://github.com/ethereum/ERCs/pull/246) (pending): Add Content-encoding handling in ERC-6944 resource request mode

## Tests structure

A ``web3://`` URL fetch is done in 3 steps:

- Parsing of the URL: extract and prepare elements for the main smartcontract call
- Calling the smartcontract: For some calldata, the smartcontract returns us bytes
- Processing of the smartcontract return: Based of various factors, an output of bytes, an HTTP code and HTTP headers are returned, ready to be displayed by the browser.

The tests groups are separated in three types : 
- ``urlParsing`` : Focus on the first step
- ``contractReturnProcessing`` : Focus on the last step
- ``fetch`` : Focus on the execution of the three steps, and their plumbing

Each file contains test groups, and as the first-level contains the ``type`` field indicating the type of these test groups.


## urlParsing tests

```
name = "..."
url = "web3://uniswap.eth"
hostDomainNameResolver = "ens"
hostDomainNameResolverChainId = 1 # If hostDomainNameResolver is non empty
contractAddress = "0x1a9C8182C09F50C8318d769245beA52c32BE35BC"
chainId = 1
resolveMode = "manual"
contractCallMode = "calldata"
calldata = "0x2f" # if contractCallMode is "calldata"
methodName = "levelAndTile" # if contractCallMode is "method"
methodArgs = [{type = "uint256"}, {type = "uint256"}] # if contractCallMode is "method"
methodArgValues = [{value = 2}, {value = 50}] # if contractCallMode is "method"
contractReturnProcessing = "decodeABIEncodedBytes"
decodedABIEncodedBytesMimeType = "text/html" # if contractReturnProcessing is "decodeABIEncodedBytes"
jsonEncodedValueTypes = [{type = 'uint256'}] # if contractReturnProcessing is "jsonEncodeValues"
```

In these tests, the input is the ``url`` field only. We are not interested in the smartcontract output, and thus we do not call the smartcontract .

These optional fields represent the expected data we extract after parsing the URL :
- ``hostDomainNameResolver`` : If any, the domain name resolver used. Possible values are : ``ens``, ``w3ns``.
  - ``hostDomainNameResolverChainId`` : If a resolver was used, this will contains the chain id of the resolver.
- ``contractAddress`` : The address of the smartcontract.
- ``chainId`` : The id of the chain where the smartcontract is located.
- ``resolveMode`` : the detected ERC-4804 resolve mode. Possible values are: ``manual``, ``auto``, ``resourceRequest``.
- ``contractCallMode`` : the way we are going to be asked to call the smartcontract. Possible values are: ``method``, ``calldata``. Depending of various factors, we may be asked to call a smartcontract method, or directly some raw calldata. 
  - If ``contractCallMode`` is ``method``, these extra fields will be used:
    - ``methodName`` : the name of the smartcontract method
    - ``methodArgs`` : an array of the types of the arguments of the smartcontract method
    - ``methodArgValues`` : an array of the arguments values
  - If ``contractCallMode`` is ``calldata``, these extra fields will be used:
    - ``calldata`` : the calldata to be sent, in hexadecimal format
- ``contractReturnProcessing`` : Indicates how the data returned by the smartcontract will be processed. Possible values are:
  - ``decodeABIEncodedBytes`` : the data returned is considered to be ABI-encoded bytes. It will be ABI-decoded and then returned. If this value is used, these extra fields will be used : 
    - ``decodedABIEncodedBytesMimeType`` : Indicates what MIME type to use when returning the bytes to the client.
  - ``jsonEncodeRawBytes`` : the data returned will be converted into an hexadecimal representation and will be JSON-encoded.
  - ``jsonEncodeValues`` : the data returned is considered to be ABI-encoded values. It will be ABI-decoded and then JSON-encoded. If this value is used, these extra fields will be used : 
    - ``jsonEncodedValueTypes`` : an array of ABI types that will be used to decode the returned data.
  - ``decodeErc5219Request`` : the data returned will be processed as specified by the ERC-5219 spec.

In case of expected error, the format will be : 
```
url = "web3://0x4e1f41613c9084fdb9e34e11fae9412427480e56/tokenHT*ML"
error = { label = "Invalid method name", httpCode = 400 }
```
- ``label`` : The message of the error. These messages are indicative, they do not need to be reproduced identical.
- ``httpCode`` : the HTTP code expected to be received by the browser.

## contractReturnProcessing tests

```
name = "..."
contractReturn = "0x0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
contractReturnProcessing = "jsonEncodeValues"
decodedABIEncodedBytesMimeType = "text/html" # if contractReturnProcessing is "decodeABIEncodedBytes"
jsonEncodedValueTypes = [{type = "bytes32"}] # If contractReturnProcessing is "jsonEncodeValues"
output = "0xfa12" # output or outputAsString
outputAsString = "[\"0x0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef\"]" # output or outputAsString
httpCode = 200
httpHeaders = {"Content-Type" = "application/json"}
```

In these tests, the input is ``contractReturn``, simulating data returned by a smartcontract, ``contractReturnProcessing`` and optionally ``decodedABIEncodedBytesMimeType`` and ``jsonEncodedValueTypes`` (see documentation above).

We are interested on testing how the smartcontract return is processed.

These fields represents the expected data we will get after the smartcontract data is processed : 
- ``output`` / ``outputAsString`` : The bytes that the browser is supposed to get and display. ``output`` is the hexadecimal representation of the bytes, and if a string representation is more legible for the test, ``outputAsString`` is the string representation.
- ``httpCode`` : The HTTP code that the browser will get and use.
- ``httpHeaders`` : The HTTP headers that the browser will get and use.

## fetch tests

```
name = "..."
url = "web3://0xA5aFC9fE76a28fB12C60954Ed6e2e5f8ceF64Ff2/levelAndTile/2/50?returns=(uint256,uint256)"
outputAsString = "[\"1\",\"36\"]"
httpCode = 200
httpHeaders = {"Content-Type" = "application/json"}
```

These tests execute all the steps, and are looking at checking the plumbing between the steps. The only input is ``url`` and the expected output are ``output``/``outputAsString``, ``httpCode`` and ``httpHeaders`` (see documentation above).
