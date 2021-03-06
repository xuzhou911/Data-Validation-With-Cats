
class: center, middle

# Data Validation with Cats

**By:** Algimantas Krasauskas

**From:** Wix.com

![cato](images/cato.gif "Cato")

---
# The Problem

We need to integrate with an external API that provides us with an email and password that both might be invalid. After this data is validated we need to use in a workflow.

```tut:silent
case class UserDTO(email: String, password: String)

def isValidEmail(email: String): Boolean = ???
def isValidPassword(password: String): Boolean = ???

```
---
# Version 0 Naive Implementation

Going the path of least resistance we choose to validate our users email and password and throw an exception when it fails.

```tut:silent
class UserValidationException extends Exception("User validation exception")

def validateUserVersion0(user: UserDTO): UserDTO =
  if (isValidEmail(user.email) && isValidPassword(user.password)) {
    user
  } else {
    throw new UserValidationException
  }
```

---
# Version 0 Naive Implementation

<img src="images/boom.gif" style="display: block; margin-left: auto;margin-right: auto; width: 75%;"/>

---
# Version 1 Smart Constructors

```tut:silent
case class Email(value: String)
object Email {
  def apply(email: String): Option[Email] = 
    Some(email).filter(isValidEmail).map(new Email(_))
}

case class Password(value: String)
object Password {
  def apply(password: String): Option[Password] = 
    Some(password).filter(isValidPassword).map(new Password(_))
}

case class User(email: Email, password: Password)

object User {
  def apply(email: Email, password: Password): User = new User(email, password)

  def fromUserDTO(user: UserDTO): Option[User] = for {
    email <- Email(user.email)
    password <- Password(user.password)
  } yield new User(email, password)
}

def validateUserVersion1(user: UserDTO): Option[User] = User.fromUserDTO(user)
```

---
#  Version 2 Transforming to Either

Informing about the correct type of failure in our computation.

```tut:silent
val userError = "User validation error"
def validateUserVersion2(user: UserDTO): Either[String, User] = 
    User.fromUserDTO(user).toRight(userError)

```
---
# Version 3 Combining Errors

```tut:silent
val emailError = "invalid email"
val passwordError = "invalid password"

def validateUserVersion3(user: UserDTO): Either[String, User] = (
    Email(user.email).toRight(emailError),
    Password(user.password).toRight(passwordError)
  ) match {
    case (Right(email), Right(password)) => Right(User(email, password))
    case (Left(error), Right(_))         => Left(error)
    case (Right(_), Left(error))         => Left(error)
    case (Left(e1), Left(e2))            => Left(e1 ++ e2)
}
```
---
# Version 4 Simplify Syntax

Using more familiar `for` comprehension syntax.

```tut:silent
def validateUserVersion4(user: UserDTO): Either[String, User] = for {
  email <- Email(user.email).toRight(emailError)
  password <- Password(user.password).toRight(passwordError)
} yield User(email, password)
```
---
# Version 5 New Syntax

```tut:silent
import cats.implicits._
import cats.data.Validated
import cats.data.Validated.{Invalid, Valid}

def validateUserVersion5(user: UserDTO): Validated[String, User] = (
  Email(user.email).toValid(emailError),
  Password(user.password).toValid(passwordError)
).mapN(User(_, _))
```
---
# Version 6 Modeling Dependent Errors

```tut:silent
import cats.data.NonEmptyList
import cats.data.ValidatedNel

sealed trait UserError
final case object PasswordValidationError extends UserError

sealed trait EmailError extends UserError
final case object InvalidEmailError extends EmailError
final case object BlackListedUserError extends EmailError

val blackListedUsers = Seq("bart@simsom.com")

def validatedEvilness(email: Email): ValidatedNel[UserError, Email] =
 Validated.condNel(!blackListedUsers.contains(email.value), 
                   email, 
                   BlackListedUserError)


def validateUserVersion6(user: UserDTO): ValidatedNel[UserError, User] = (
  Email(user.email).toValidNel(InvalidEmailError)
                   .andThen(validatedEvilness),
  Password(user.password).toValidNel(PasswordValidationError)
).mapN(User(_, _))
```
---

# Version 7 Generalizing

```tut:silent
def validateUserVersion1(user: UserDTO) = validateUserVersion6(user).toOption

def validateUserVersion3(user: UserDTO) = validateUserVersion6(user).toEither
```

---
# Additional information

Inspired by: https://youtu.be/P8nGAo3Jp-Q @DanielaSfregola

Additional keywords: ApplicativeError, MonadError

Github: https://github.com/Algiras/Data-Validation-With-Cats

Binder: https://bit.ly/31H97KW

---

# Questions?

<img src="images/questions.gif" style="display: block; margin-left: auto;margin-right: auto; width: 75%;"/>