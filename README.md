# Explainer for Asynchronous Creation and Availability Checks for Text Detection API

This proposal is an early design sketch by Chrome Device API team to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.

## Participate
- https://github.com/explainers-by-googlers/async-textdetector/issues
- https://github.com/WICG/shape-detection-api/issues/109


## Motivation

The current Text Detection API provides a synchronous constructor: new TextDetector().

This simple, synchronous design does not account for the complex initialization that may be required by underlying Optical Character Recognition (OCR) engines. Many modern, high-quality OCR engines rely on language-specific resources or other dependencies that are not always bundled with the browser.

This presents three major challenges for web developers:

1. Unknown Language Support: There is no way for a developer to programmatically check if the user's browser can detect text in a specific language (e.g., Japanese, Arabic, Russian) before attempting to use the API.

2. Opaque Initialization Process: If the browser needs to download and initialize a language resource, a process which may not be immediate, this process is completely opaque to the developer. It can lead to a "janky" or unresponsive-seeming user experience where the first detect() call is slow to start. Lacking an asynchronous creation method, the developer cannot prevent this "janky" experience.

3. No Cancellation Support: If this hidden initialization takes too long, the developer has no way to cancel it. This can lead to a poor experience if the user navigates away or tries to perform a different action.

This proposal aims to create a more robust and predictable API that gives developers the control they need to build high-quality experiences.

## Proposed Solution

To address these issues, we propose enhancing the TextDetector interface by adopting an asynchronous design, similar to other modern web APIs that handle complex, time-consuming initialization. This availability() and create() pattern is consistent with other proposals that manage complex, on-demand resources, such as the Writing Assistance API and the Translator and Language Detector API.

The proposed changes introduce two new static methods:

1. TextDetector.availability(options): This static method allows a website to quickly check if the user agent supports text detection for a given set of languages. It returns a status indicating whether the resources are supported by the user agent ("available") or not ("unavailable"). This check does not reveal whether the resources are already downloaded, protecting user privacy.

2. TextDetector.create(options): This new, Promise-based factory method replaces the synchronous constructor. It handles the entire setup process, including the asynchronous download (if necessary) and initialization of any required resources. It also provides mechanisms for cancellation (AbortSignal), allowing for a much-improved user experience.

These additions will make the Text Detection API significantly more powerful, easier for developers to use correctly, and easier for browser vendors to implement with powerful, on-demand OCR engines.

## Detailed Design and Usage Examples

### 1. Checking for Language Availability

Developers can first check if the language(s) they need are supported.

```js
const status = await TextDetector.availability({ languages: ['ar'] });

if (status === 'available') {
  console.log('Arabic support is available.');
} else {
  console.log('Arabic support is not available.');
}
```

### 2. Creating a TextDetector (Asynchronously)

The new factory pattern handles the resource-loading (which may or may not involve a download) and allows for cancellation.

```js
const controller = new AbortController();

// Example: Abort if it takes longer than 15 seconds
const timeoutId = setTimeout(() => controller.abort(), 15000);

try {
  const detector = await TextDetector.create({
    languages: ['ja'],
    signal: controller.signal
  });
  clearTimeout(timeoutId);
  console.log('Japanese TextDetector is ready.');
  const texts = await detector.detect(imageElement);
  console.log(texts);
} catch (err) {
  if (err.name === 'AbortError') {
    console.log('TextDetector creation was cancelled (timeout).');
  } else {
    console.error('Failed to create the detector:', err);
  }
}
```

## Proposed IDL Changes

Below is the proposed IDL for the enhanced TextDetector interface.

```webidl
[Exposed=Window, SecureContext]
interface TextDetector {
  // Static method to check for language support and resource availability.
  static Promise<AvailabilityStatus> availability(optional TextDetectorOptions options = {});

  // Static factory method to create a configured TextDetector instance.
  static Promise<TextDetector> create(optional TextDetectorCreateOptions options = {});

  // Existing instance method.
  Promise<sequence<DetectedText>> detect(ImageBitmapSource image);
};

// --- Availability Checking ---

enum AvailabilityStatus {
  "unavailable",
  "available"
};

// --- Instance Creation and Core Options ---

// Base options for creating a TextDetector or checking availability.
// The `languages` sequence should contain strings conforming to BCP 47 language codes
// (e.g., "en-US", "ja", "ar").
dictionary TextDetectorOptions {
  // Optional: A hint for the expected language(s) in the image.
  sequence<DOMString> languages;
};

// Extends the core options with support for cancellation.
dictionary TextDetectorCreateOptions : TextDetectorOptions {
  AbortSignal signal;
};
```

## Security and Privacy Considerations

This proposal introduces one new potential privacy consideration: fingerprinting.

The TextDetector.availability() method reveals information about the language detection capabilities of the user's browser. A site could probe for a wide variety of languages (ja, ar, ko, ru, hi, etc.) and use the list of "available" capabilities as an entropy source to help identify a unique user.

Mitigations:

* This design intentionally omits a "downloadable" or similar state. By only exposing "available" and "unavailable", the API avoids the primary fingerprinting vector: user state (i.e., whether a resource is already cached or downloaded). The availability() method only reports the static capabilities of the browser's underlying text detection implementation, not the user's history. This means that all users with the same browser version and implementation will report the same set of supported languages, making it a poor tool for fingerprinting and- significantly reducing the privacy risk.

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Matt Reynolds
- Reilly Grant
- Domenic Denicola
