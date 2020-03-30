# Firebase Phone Authentication using Firebase

Demo Project to show how to do Phone Authentication in Flutter using Firebase backend.

## Motivation

I felt there is no setp-by-step documentation for Firebase Phone Authentication in Flutter. There were couple of implementations on Medium but are out-dated and one article got it but that failed to explain some crucial concepts. AndroidX brought a lot of changes in addition to FlutterFire plugins update. I asked this question on [stackoverflow](https://stackoverflow.com/questions/50181000/how-to-do-phone-authentication-in-flutter-using-firebase/56823147#56823147) almost 2 years ago, but failed to get step-by-step answer. So I created this demo project.

## Setup Firebase with Flutter

You can setup Firebase by following this guide: https://firebase.google.com/docs/flutter/setup

If you encounter errors with `MultiDex` or `AndroidX` make sure you follow below steps

```gradle
// Inside android folder in app level build.gradle file
...
android {
  ...
  defaultConfig {
    ...
    multiDexEnabled true // add this
  }
  ...
}
...
dependencies {
  // implementation 'com.android.support:multidex:1.0.3' // No AndroidX support
  implementation 'com.android.support:multidex:2.0.1' // Add this to support AndroidX
  ...
}
...
```

## Steps

1. Ask for user's phoneNumber
2. Get OTP from Firebase
3. SignIn to Firebase

## Rules

- SignIn/Login is done in the same way.
- The OTP is only used to get `AuthCrendential` object
- `AuthCredential` object is the only thing that is used to signIn the user.
    It is obtained either from `verificationCompleted` callback function in `verifyPhoneNumber` or from the `PhoneAuthProvider`.
(Don't worry if it's confusing, keep reading, you'll get it)

## Workflow

1. User gives the `phoneNumber`
2. Firebase sends OTP
3. SignIn the user
    - If the SIM card with the `phoneNumber` is not in the device that is currently running the app,
        - We have to first ask the OTP and get `AuthCredential` object
        - Next we can use that `AuthCredential` to signIn
       This method works even if the `phoneNumber` is in the device
    - Else if user provided SIM phoneNumber is in the device running the app,
        - We can signIn without the OTP.
        - because the `verificationCompleted` callback from `submitPhoneNumber` function gives the `AuthCredential` object which is needed to signIn the user
        - But in the previous case it ain't called because the SIM is not in the phone.

## Functions

- SubmitPhoneNumber

```dart
Future<void> _submitPhoneNumber() async {
    /// NOTE: Either append your phone number country code or add in the code itself
    /// Since I'm in India we use "+91 " as prefix `phoneNumber`
    String phoneNumber = "+91 " + _phoneNumberController.text.toString().trim();
    print(phoneNumber);

    /// The below functions are the callbacks, separated so as to make code more redable
    void verificationCompleted(AuthCredential phoneAuthCredential) {
      print('verificationCompleted');
      ...
      this._phoneAuthCredential = phoneAuthCredential;
      print(phoneAuthCredential);
    }

    void verificationFailed(AuthException error) {
      ...
      print(error);
    }

    void codeSent(String verificationId, [int code]) {
      ...
      print('codeSent');
    }

    void codeAutoRetrievalTimeout(String verificationId) {
      ...
      print('codeAutoRetrievalTimeout');
    }

    await FirebaseAuth.instance.verifyPhoneNumber(
      /// Make sure to prefix with your country code
      phoneNumber: phoneNumber,

      /// `seconds` didn't work. The underlying implementation code only reads in `millisenconds`
      timeout: Duration(milliseconds: 10000),

      /// If the SIM (with phoneNumber) is in the current device this function is called.
      /// This function gives `AuthCredential`. Moreover `login` function can be called from this callback
      verificationCompleted: verificationCompleted,

      /// Called when the verification is failed
      verificationFailed: verificationFailed,

      /// This is called after the OTP is sent. Gives a `verificationId` and `code`
      codeSent: codeSent,

      /// After automatic code retrival `tmeout` this function is called
      codeAutoRetrievalTimeout: codeAutoRetrievalTimeout,
    ); // All the callbacks are above
  }
```

- SubmitOTP

```dart
void _submitOTP() {
    /// get the `smsCode` from the user
    String smsCode = _otpController.text.toString().trim();

    /// when used different phoneNumber other than the current (running) device
    /// we need to use OTP to get `phoneAuthCredential` which is inturn used to signIn/login
    this._phoneAuthCredential = PhoneAuthProvider.getCredential(
        verificationId: this._verificationId, smsCode: smsCode);

    _login();
  }
```

- SignIn/LogIn

```dart
Future<void> _login() async {
    /// This method is used to login the user
    /// `AuthCredential`(`_phoneAuthCredential`) is needed for the signIn method
    /// After the signIn method from `AuthResult` we can get `FirebaserUser`(`_firebaseUser`)
    try {
      await FirebaseAuth.instance
          .signInWithCredential(this._phoneAuthCredential)
          .then((AuthResult authRes) {
        _firebaseUser = authRes.user;
        print(_firebaseUser.toString());
      });
      ...
    } catch (e) {
      ...
      print(e.toString());
    }
  }
```

- Logout

```dart
  Future<void> _logout() async {
    /// Method to Logout the `FirebaseUser` (`_firebaseUser`)
    try {
      // signout code
      await FirebaseAuth.instance.signOut();
      _firebaseUser = null;
      ...
    } catch (e) {
      ...
      print(e.toString());
    }
  }
```

For more details on implementation please refer to the `lib/main.dart` file.

## Contributions

- Please feel free to contribute to this README (Pull Requests) and this [stackoverflow answer](https://stackoverflow.com/questions/50181000/how-to-do-phone-authentication-in-flutter-using-firebase/#52912441)
- If you find this `README` useful please star the repository
