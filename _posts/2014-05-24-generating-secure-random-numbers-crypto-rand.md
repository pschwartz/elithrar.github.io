---
layout: post
title: Generating Secure Random Numbers Using crypto/rand
categories: go, security
---

You're writing an application and you need to generate some session keys, CSRF tokens, and HMACs. For all of these activities, you need a crytographically secure pseudo-random number generator (CSPRNG). Or, in other words, a source of random bytes that is unpredictable and without bias. Specifically, you want to mitigate the chance that an attacker can predict future tokens based on previous tokens.

So what does this mean? No `math.Rand`, no `time.UnixNano`; in fact, it means that you only (ever) use the system CSPRNG that your operating system provides. This means using [/dev/urandom/](http://sockpuppet.org/blog/2014/02/25/safely-generate-random-numbers/) and Windows' CryptGenRandom API.

Go's `crypto/rand` package, thankfully, abstracts these implementation details away to minimise the risk of getting it wrong:

```go

import(
	"crypto/rand"
	"encoding/base64"
)

// GenerateRandomBytes returns securely generated random bytes. 
// It will return an error if the system's secure random
// number generator fails to function correctly, in which
// case the caller should not continue.
func GenerateRandomBytes(n int) ([]byte, error) {
	b := make([]byte, n)
	_, err := rand.Read(b)
    // Note that err == nil only if we read len(b) bytes.
	if err != nil {
		return nil, err
	}

	return b, nil
}

// GenerateRandomString returns a URL-safe, base64 encoded
// securely generated random string.
// It will return an error if the system's secure random
// number generator fails to function correctly, in which
// case the caller should not continue.
func GenerateRandomString(s int) (string, error) {
	b, err := GenerateRandomBytes(s)
	return base64.URLEncoding.EncodeToString(b), err
}

// Example: this will give us a 44 byte, base64 encoded output
token, err := GenerateRandomString(32)
if err != nil {
	// Serve an appropriately vague error to the
    // user, but log the details internally.
}
```

We have two functions here:

* `GenerateRandomBytes` is useful when we need the raw bytes for another cryptographic function, such as HMAC keys.
* `GenerateRandomString` wraps this and generates keys we can use for session IDs, CSRF tokens and the like. We base64 URL encode the output in order to provide secure strings that we can use in file-names, templates, HTTP headers and to minimise encoding overhead compared to hex encoding.

For CSRF tokens and session [cookie] IDs, 32 bytes (256 bits) is more than sufficient and common in a number of large web frameworks. If you're generating HMAC keys, you'll need to size it based on [which HMAC algorithm](http://tools.ietf.org/html/rfc4868#section-2.6) you are using, and the same goes for [AES](http://golang.org/pkg/crypto/aes/#NewCipher). 

Note that, critically, we always ensure that we check for the *extremely* rare case our operating systems CSPRNG might fail us with an error. And if it's failing, we want to fail too, because that means there's something seriously wrong with our operating system. Checking and handling errors is also idiomatic Go, and good software practice in general. In this case however it's definitely worth emphasising the need to both log the error and stop processing.

If you are interested in the mechanics behind CSPRNGs I'd recommend reading Schneier's [Crytography Engineering](http://www.amazon.com/gp/product/0470474246/ref=as_li_tl?ie=UTF8&camp=1789&creative=390957&creativeASIN=0470474246&linkCode=as2&tag=eatsleeprepea-20&linkId=64MI7XJMN33KD6AB), and if you're interested in web security in general, [Adam Langley's blog](https://www.imperialviolet.org/) is worth following—noting that Adam contributes to much of Go's crytographic code.
