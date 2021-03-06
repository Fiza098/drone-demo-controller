# Drone Demo Controller
iOS app for controlling Chronicled's drone demo

### Description
This app verifies Chronicled BLE chips using Apple's [CoreBluetooth](https://developer.apple.com/library/content/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/AboutCoreBluetooth/Introduction.html) library. This app also controls the demo's [lights](http://www2.meethue.com/en-us/about-hue?&origin=%7Cmckv%7CsC8ExIQm5_dc&pcrid=78454527516%7Cplid%7C&gclid=Cj0KEQjw1ee_BRD3hK6x993YzeoBEiQA5RH_BBB40tvaoMY9aw0S3Imvvsec8mek3GTGaNahySqvuEUaAovE8P8HAQ) and [drapes](http://www.lutron.com/en-US/Products/Pages/ShadingSystems/SivoiaQS/WindowTreatments/DraperySystems/Drapery.aspx)

### Demo Explaination

- Done approaches iPhone near drapes to be opened
- This app discovers the chip and requests a challenge from the server. The identity used is the `deviceID`. A deviceID is a 6 byte unique hex string. `ble:1.0:1234567890ab`

```swift
 static func requestChallenge(identity: String, cb: (Response<AnyObject, NSError> -> ())) {
     let req = Alamofire.request(.POST,
                                 "\(Config.domain)requestChallenge",
                                 parameters: ["identity" : identity],
                                 encoding: .JSON,
                                 headers: headers)

     req.responseJSON(completionHandler: cb)
 }
```

- If the Registrant has access to the door, a challenge will be returned from the server. If the Registrant doesn't have access, the chip will be rejected. The app will connect to the chip if a challenge is returned: 

```swift
self.centralManager?.connectPeripheral(peripheral, options: nil)
```

- After connecting to the chip, the app writes the challenge received from the server to the Challenge Characteristic of the BLE chip

```swift
peripheral.writeValue(challenge.dataWithHexString(),
                      forCharacteristic: challengeCharecteristic,
                      type: .WithResponse)
```

- Now the app waits for the signature to be generated by the chip. The signature will be written to the Signature characteristic. After the signature has been generated, the app reads the signature.

```swift
peripheral.readValueForCharacteristic(self.signatureChararacteristic)
```

- The app then sends the signature, challenge, and identity to the verification endpoint.

```swift
static func sendVerification(identity: String,
                             challenge: String,
                             signature: String,
                             cb: (Response<AnyObject, NSError> -> ())) {
    let params = [
        "identity" : identity,
        "challenge" : challenge,
        "signature" : signature
    ]

    let req = Alamofire.request(.POST,
                                "\(Config.domain)verifyChallenge",
                                parameters: params,
                                encoding: .JSON,
                                headers: headers)

    req.responseJSON(completionHandler: cb)
}
```
- The server will return whether or not the signature is verified. Using this response, the app will either reject or grand acccess to the chip.
