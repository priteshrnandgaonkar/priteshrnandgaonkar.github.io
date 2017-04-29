---
layout: post
title: Unit-Testing a feature
---

Unit-Testing is an important tool to validate the functionality. But I feel that it is not used to its full potential. Usually developers just test out the functions which takes in some arguments and returns some data model. Although such tests are usefull but they can't help you to ascertain that the feature would definitely work if the tests are successfull.

So lets try to unit test the login feature. Login is usually present in all applications and writing unit tests to test out this feature would be quite usefull.

Have a look at the below Login Screen

![Screen](https://priteshrnandgaonkar.github.io/assets/Blogs/unit-testing-a-feature/LoginScreen.png)

The requirement for this Login is as follows:-

* The UserName should have atleast 2 and atmost 10 characters
* Password should be of atleast 3 characters with atleast one upercase letter and one decimal digit

Now our plan is to not only test the above validation but also the end-to-end flow. We would test the following points

* Login API failures like:-
	* Invalid credentials
	* Nil Data
	* Wrong Status code
* Validation of the input fields

### Architecting the Feature

The main idea to test out the above feature would be dependency-injection. We would have seperate `LoginService` class that handles the api request and validation. By doing this, while testing, we can create its fake and test out our scenario in different cases like wrong input credentials, unexpected response from the api etc. We would explore our classes in detail in the following paras.

I would have a `ViewController` which looks like the following

``` swift
import UIKit

class ViewController: UIViewController {

    @IBOutlet weak var username: UITextField!
    @IBOutlet weak var password: UITextField!
    var loginService: LoginService!
    @IBOutlet weak var loginButton: UIButton!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        loginService = LoginServiceHandler(delegate: self)
    }
    
    @IBAction func tappedLogin(_ sender: UIButton) {
        loginService.login(withUsername: username.text, password: password.text)
    }
}

extension ViewController: LoginServiceDelegate {
    func loginSuccessfull(withUser user: User) {
        showAlert(withTitle: "Success", message: "Hello \(user.name)")
    }
    
    func handle(error: Error) {
        showAlert(withTitle: "Error", message: error.localizedDescription)
    }
}

``` 

You can see that `ViewController` is very small and lot of the code in it is self explanatory. 

The important variable in the above `ViewController` is `loginService `. This variable is of type `LoginService` which is responsible for validating the passed input and consequently hitting the login api. It takes a `delegate` as an argument to instantiate. The delegate will get callbacks if the login api was successfull in returning the `User` data. It would also get callback if there happen to be some error in passed arguments or in the response from the api

``` swift
protocol LoginService {
    var delegate: LoginServiceDelegate { get }
    func login(withUsername username: String?, password: String?)
}

// Delegate for the LoginService
protocol LoginServiceDelegate {
    
    func loginSuccessfull(withUser user: User)
    
    func handle(error: Error)
}


// The class is responsible to validate the input 
// parameters and consequently hit the login api
class LoginServiceHandler: LoginService {
    let delegate: LoginServiceDelegate
    
    init(delegate: LoginServiceDelegate) {
        self.delegate = delegate
    }
    
    func login(withUsername username: String?, password: String?) {
        
        // hit login service
        let result = validate(userName: username ?? "", password: password ?? "")
        switch result {
        case .failure(let error):
            //handle error
            delegate.handle(error: error)
            return
        case .success(_):
            // hit login api
            break
        }
        
        guard let username = username else {
            delegate.handle(error: LoginFormValidationError.invalidUsernameLength)
            return
        }
        // Actually you would hit the login api here,
        // I was too lazy for that, thus I returned
        // the User object instead
        let user = User(name: username, userName: username, email: "\(username)@gmail.com", address: "Find me if you can", designation: "Software Developer")
        delegate.loginSuccessfull(withUser: user)
        
    }
    
    
    func validate(userName username: String, password: String) -> Result<Bool> {
        guard username.characters.count >= 2 && username.characters.count <= 10 else {
            return Result.failure(LoginFormValidationError.invalidUsernameLength)
        }
        
        guard password.characters.count > 2 else {
            return Result.failure(LoginFormValidationError.invalidPasswordLength)
        }
        
        let capitalLetterRegEx  = ".*[A-Z]+.*"
        let texttest = NSPredicate(format:"SELF MATCHES %@", capitalLetterRegEx)
        let capitalresult = texttest.evaluate(with: password)
        
        let numberRegEx  = ".*[0-9]+.*"
        let texttest1 = NSPredicate(format:"SELF MATCHES %@", numberRegEx)
        let numberresult = texttest1.evaluate(with: password)
        
        guard capitalresult && numberresult else {
            return Result.failure(LoginFormValidationError.invalidPasswordCharacters)
        }
        
        return .success(true)
    }
}

```

From the above code snippets you would have understand how our entire functionality of the feature works.

So lets try out some test cases.

One test case would be to test for the nil data when the api is hit

``` swift
class LoginApplicationTests: XCTestCase {
    var vcLogin: ViewController!
    
    override func setUp() {
        super.setUp()
        
        let storyboard = UIStoryboard(name: "Main", bundle: nil)
        let vc: ViewController = storyboard.instantiateViewController(withIdentifier: "ViewController") as! ViewController
        vcLogin = vc      
        _ = vcLogin.view // To call viewDidLoad
    }

 	func testDataNilInLoginServiceEndToEnd() {
        
        let actionDelegate = FakeLoginActionService()
        
        vcLogin.username.text = "pritesh"
        vcLogin.password.text = "poY7trl"
        vcLogin.loginService = FakeFailureLoginService(error: .nilData, delegate: actionDelegate)
        vcLogin.loginButton.sendActions(for: .touchUpInside)
        guard let receivedError = actionDelegate.error as? LoginServiceError else {
            XCTFail("Expected error of type LoginFormValidationError but got \(actionDelegate.error)")
            return
        }
        XCTAssertTrue(actionDelegate.isHandleErrorCalled , "The function handleError is not called.")
        XCTAssert(receivedError == LoginServiceError.nilData, "The function handleError is called but the error received as the argument in the function is wrong, Expected the error of type \(LoginServiceError.nilData) but got \(receivedError)")
    }
}
```

You can see that that the viewcontroller is set up in the `setUp()` function. This `setUp()` function is called before the execution of the every tests. Now in the `testDataNilInLoginServiceEndToEnd()` function, I set the required input parameters in the `vcLogin` which is a viewcontroller. You would have noticed `FakeLoginActionService` and `FakeFailureLoginService`. This are the fakes which I have used to test out the `nilData` testcase

`FakeFailureLoginService` conforms `LoginService` and instantiates with an error and delegate. The error is used to call a delegate functions(which is of type `LoginServiceDelegate`) `handle(error: Error)` function.

`FakeLoginActionService` conforms `LoginServiceDelegate` and it has the variables like

``` swift
    var isLoginSuccessFullCalled = false
    var isHandleErrorCalled = false
    var error: Error? = nil
```

The `FakeLoginActionService` and `FakeFailureLoginService` are as follows

``` swift

class FakeLoginActionService: LoginServiceDelegate {
    
    var isLoginSuccessFullCalled = false
    var isHandleErrorCalled = false
    var error: Error? = nil
    
    func loginSuccessfull(withUser user: User) {
        isLoginSuccessFullCalled = true
    }
    
    func handle(error: Error) {
        isHandleErrorCalled = true
        self.error = error
    }
}

class FakeFailureLoginService: LoginService {
    let error: LoginServiceError
    var delegate: LoginServiceDelegate
    
    init(error: LoginServiceError, delegate: LoginServiceDelegate) {
        self.error = error;
        self.delegate = delegate
    }
    func login(withUsername username: String?, password: String?) {
        delegate.handle(error: error)
    }
}

```

Going back to the test function,

``` swift

    func testDataNilInLoginServiceEndToEnd() {
        
        let actionDelegate = FakeLoginActionService()
        
        vcLogin.username.text = "pritesh"
        vcLogin.password.text = "poY7trl"
        vcLogin.loginService = FakeFailureLoginService(error: .nilData, delegate: actionDelegate)
        vcLogin.loginButton.sendActions(for: .touchUpInside)
        guard let receivedError = actionDelegate.error as? LoginServiceError else {
            XCTFail("Expected error of type LoginFormValidationError but got \(actionDelegate.error)")
            return
        }
        XCTAssertTrue(actionDelegate.isHandleErrorCalled , "The function handleError is not called.")
        XCTAssert(receivedError == LoginServiceError.nilData, "The function handleError is called but the error received as the argument in the function is wrong, Expected the error of type \(LoginServiceError.nilData) but got \(receivedError)")
    }

```

I assigned the input argumets to text field by `        vcLogin.username.text = "pritesh"` `vcLogin.password.text = "poY7trl"` . I also assigned the fake login service class to return error of type `nilData`, when the user taps Login button. But how you would simulate the button tap. For that I have used
`        vcLogin.loginButton.sendActions(for: .touchUpInside)
`
There's a function available in `UIControl`  which sends action. The documentation reads as follows

``` swift
 open func sendActions(for controlEvents: UIControlEvents) 
 // send all actions associated with events

```

Now `        vcLogin.loginButton.sendActions(for: .touchUpInside)
` calls `@IBAction func tappedLogin(_ sender: UIButton)` of the `vcLogin`. This function calls the LoginService's `login(withUsername username: String?, password: String?)` which always fails(as we have assigned to the fake one) and calls the LoginServiceDelegate's `handle(error: Error)`, where we test whether the error with which it is called is of the expected type or not. We also test whether the function is called or not. The assertions are as follows

``` swift
guard let receivedError = actionDelegate.error as? LoginServiceError else {
    XCTFail("Expected error of type LoginFormValidationError but got \(actionDelegate.error)")
    return
}
XCTAssertTrue(actionDelegate.isHandleErrorCalled , "The function handleError is not called.")
XCTAssert(receivedError == LoginServiceError.nilData, "The function handleError is called but the error received as the argument in the function is wrong, Expected the error of type \(LoginServiceError.nilData) but got \(receivedError)")
```

The above assertions test whether the `LoginServiceDelegate` receives the error of type `LoginServiceError `. It also test if the function `handle(error)` is called or not. Later it tests if the error is of type `.nilData` or not.

I think you would have understood, how and what this function is testing out. The similar idea is used to test the other points mentioned in the above para.

``` swift

func testInvalidUsernameLengthEndToEnd() {
    
    let actionDelegate = FakeLoginActionService()

    vcLogin.username.text = "p"
    vcLogin.loginService = LoginServiceHandler(delegate: actionDelegate)
    vcLogin.loginButton.sendActions(for: .touchUpInside)
    guard let receivedError = actionDelegate.error as? LoginFormValidationError else {
        XCTFail("Expected error of type LoginFormValidationError but got \(actionDelegate.error)")
        return
    }
    XCTAssertTrue(actionDelegate.isHandleErrorCalled , "The function handleError is not called.")
    XCTAssert(receivedError == LoginFormValidationError.invalidUsernameLength, "The function handleError is called but the error received as the argument in the function is wrong, Expected the error of type \(LoginFormValidationError.invalidUsernameLength) but got \(receivedError)")
}

 func testInvalidPasswordLengthEndToEnd() {
        
        let actionDelegate = FakeLoginActionService()
        
        vcLogin.username.text = "pritesh"
        vcLogin.password.text = "po"
        vcLogin.loginService = LoginServiceHandler(delegate: actionDelegate)
        vcLogin.loginButton.sendActions(for: .touchUpInside)
        guard let receivedError = actionDelegate.error as? LoginFormValidationError else {
            XCTFail("Expected error of type LoginFormValidationError but got \(actionDelegate.error)")
            return
        }
        XCTAssertTrue(actionDelegate.isHandleErrorCalled , "The function handleError is not called.")
        XCTAssert(receivedError == LoginFormValidationError.invalidPasswordLength, "The function handleError is called but the error received as the argument in the function is wrong, Expected the error of type \(LoginFormValidationError.invalidPasswordLength) but got \(receivedError)")
    }
    
func testInvalidCredentialsInLoginServiceEndToEnd() {
    
    let actionDelegate = FakeLoginActionService()
    
    vcLogin.username.text = "pritesh"
    vcLogin.password.text = "poY7trl"
    vcLogin.loginService = FakeFailureLoginService(error: .invalidCredentials, delegate: actionDelegate)
    vcLogin.loginButton.sendActions(for: .touchUpInside)
    guard let receivedError = actionDelegate.error as? LoginServiceError else {
        XCTFail("Expected error of type LoginFormValidationError but got \(actionDelegate.error)")
        return
    }
    XCTAssertTrue(actionDelegate.isHandleErrorCalled , "The function handleError is not called.")
    XCTAssert(receivedError == LoginServiceError.invalidCredentials, "The function handleError is called but the error received as the argument in the function is wrong, Expected the error of type \(LoginServiceError.invalidCredentials) but got \(receivedError)")
}
    
func testInvalidPasswordCharactersEndToEnd() {
    
    let actionDelegate = FakeLoginActionService()
    
    vcLogin.username.text = "pritesh"
    vcLogin.password.text = "poytrl"
    vcLogin.loginService = LoginServiceHandler(delegate: actionDelegate)
    vcLogin.loginButton.sendActions(for: .touchUpInside)
    guard let receivedError = actionDelegate.error as? LoginFormValidationError else {
        XCTFail("Expected error of type LoginFormValidationError but got \(actionDelegate.error)")
        return
    }
    XCTAssertTrue(actionDelegate.isHandleErrorCalled , "The function handleError is not called.")
    XCTAssert(receivedError == LoginFormValidationError.invalidPasswordCharacters, "The function handleError is called but the error received as the argument in the function is wrong, Expected the error of type \(LoginFormValidationError.invalidPasswordCharacters) but got \(receivedError)")
}
    
func testDataNilInLoginServiceEndToEnd() {
    
    let actionDelegate = FakeLoginActionService()
    
    vcLogin.username.text = "pritesh"
    vcLogin.password.text = "poY7trl"
    vcLogin.loginService = FakeFailureLoginService(error: .nilData, delegate: actionDelegate)
    vcLogin.loginButton.sendActions(for: .touchUpInside)
    guard let receivedError = actionDelegate.error as? LoginServiceError else {
        XCTFail("Expected error of type LoginFormValidationError but got \(actionDelegate.error)")
        return
    }
    XCTAssertTrue(actionDelegate.isHandleErrorCalled , "The function handleError is not called.")
    XCTAssert(receivedError == LoginServiceError.nilData, "The function handleError is called but the error received as the argument in the function is wrong, Expected the error of type \(LoginServiceError.nilData) but got \(receivedError)")
}
    
func testWrongStatusCodeInLoginServiceEndToEnd() {
    
    let actionDelegate = FakeLoginActionService()
    
    vcLogin.username.text = "pritesh"
    vcLogin.password.text = "poY7trl"
    vcLogin.loginService = FakeFailureLoginService(error: .wrongStatusCode, delegate: actionDelegate)
    vcLogin.loginButton.sendActions(for: .touchUpInside)
    guard let receivedError = actionDelegate.error as? LoginServiceError else {
        XCTFail("Expected error of type LoginFormValidationError but got \(actionDelegate.error)")
        return
    }
    XCTAssertTrue(actionDelegate.isHandleErrorCalled , "The function handleError is not called.")
    XCTAssert(receivedError == LoginServiceError.wrongStatusCode, "The function handleError is called but the error received as the argument in the function is wrong, Expected the error of type \(LoginServiceError.wrongStatusCode) but got \(receivedError)")
}
    
```

The following code snippet tests the validation logic

``` swift

func testSuccessfullValidation() {
    let actionDelegate = FakeLoginActionService()

    let loginService = LoginServiceHandler(delegate: actionDelegate)
    loginService.login(withUsername: "pritesh", password: "aK8oio")
    
    if !actionDelegate.isLoginSuccessFullCalled {
        XCTFail("Function validate returned error whereas it was expected to succeed. The error is \(actionDelegate.error?.localizedDescription)")
    }
    
    XCTAssertTrue(actionDelegate.isLoginSuccessFullCalled , "The function loginsuccessfull is not called.")
    
}

func testUsernameLengthValidation() {
    
    let actionDelegate = FakeLoginActionService()
    
    let loginService = LoginServiceHandler(delegate: actionDelegate)
    loginService.login(withUsername: "p", password: "aK8oio")
    
    XCTAssertTrue(actionDelegate.isHandleErrorCalled , "The function handle error is not called.")
    
    guard let loginFormError = actionDelegate.error as? LoginFormValidationError else {
        XCTFail("Expected error of type LoginFormValidationError but got \(actionDelegate.error)")
        return
    }

    XCTAssert(loginFormError == LoginFormValidationError.invalidUsernameLength, "Expected validation error of type invalidUsernameLength but got \(loginFormError)")

}

func testPasswordLengthValidation() {
    
    let actionDelegate = FakeLoginActionService()
    
    let loginService = LoginServiceHandler(delegate: actionDelegate)
    loginService.login(withUsername: "pritesh", password: "a")
    
    XCTAssertTrue(actionDelegate.isHandleErrorCalled , "The function handle error is not called.")
    
    guard let loginFormError = actionDelegate.error as? LoginFormValidationError else {
        XCTFail("Expected error of type LoginFormValidationError but got \(actionDelegate.error)")
        return
    }
    
    XCTAssert(loginFormError == LoginFormValidationError.invalidPasswordLength, "Expected validation error of type invalidUsernameLength but got \(loginFormError)")

}

func testPasswordCharacterValidation() {
    
    let actionDelegate = FakeLoginActionService()
    
    let loginService = LoginServiceHandler(delegate: actionDelegate)
    loginService.login(withUsername: "pritesh", password: "abjkop")
    
    XCTAssertTrue(actionDelegate.isHandleErrorCalled , "The function handle error is not called.")
    
    guard let loginFormError = actionDelegate.error as? LoginFormValidationError else {
        XCTFail("Expected error of type LoginFormValidationError but got \(actionDelegate.error)")
        return
    }
    
    XCTAssert(loginFormError == LoginFormValidationError.invalidPasswordCharacters, "Expected validation error of type invalidUsernameLength but got \(loginFormError)")

}
    

```

You can look at the code [here](https://github.com/priteshrnandgaonkar/TestLoginApplication). You can give feedback below.
