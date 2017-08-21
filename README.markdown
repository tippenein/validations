validations
===========

table of contents
-----------------

[what is "validations"?](#what-is-validations)

[existing solutions, and their
problems](#existing-solutions-and-their-problems)

    [smart constructors](#smart-constructors)

    [digestive-functors formlet
style](#digestive-functors-formlet-style)

[ideas behind the structure of
"validations"](#ideas-behind-the-structure-of-validations)

[Example](#example)

[testing](#testing)

[digestive-functors integration](#digestive-functors-integration)

What is "validations"?
----------------------

**validations** is a Haskell library that provides a flexible,
composable way to define validations of a domain model.

existing solutions, and their problems
--------------------------------------

There is a number of ways to do domain model validation in Haskell, but
each current method has drawbacks. Let's imagine a simple user model:

```haskell
data User = User
  { _firstName    :: Text
  , _lastName     :: Text
  , _emailAddress :: Text
  } deriving Show
```

We want to check that the first name is not empty and starts with the
letter A, that the last name is not empty, that the email address is not
empty, and that the email address is confirmed by a value that isn't
stored in User. We also want all checkers to conform to the type

```haskell
a -> Either e b
```

or

```haskell
(Monad m) => a -> m (Either a b)
```

.

So, our checkers could look something like

```haskell
notEmpty :: (Monoid a, Eq a) => a -> Either Text a
notEmpty x =
  if (x == mempty)
     then Left "is empty"
     else Right x
```

,

```haskell
startsWith :: Text -> Text -> Either Text Text
startsWith predicate input = 
  if predicate `isPrefixOf` input
     then Right input
     else Left $ "does not start with " <> predicate
```

, and

```haskell
confirms :: (Eq a) => a -> a -> Either Text a
confirms a b = case (a == b) of
  True -> Right b
  False -> Left "fields do not match."
```

.

### smart constructors ###

The simplest way to do this is with a smart constructor:

```haskell
user :: Text -> Text -> Text -> Text -> Either Text User
user firstName lastName emailAddress emailAddressConfirm = do
  firstName'    <- notEmpty firstName >>= startsWith "A"
  lastName'     <- notEmpty lastName
  emailAddress' <- notEmpty emailAddress
  confirmed     <- emailAddressConfirm `confirms` emailAddress'
  return $ User {_firstName = firstName', _lastName = lastName', _emailAddress = confirmed }
```

This will enforce all of our invariants, but there's a problem. If any
of our validations fail, then we only get the results of the failure of
first validation. If firstName and lastName are both empty, we'd like to
know that the validation logic failed for both. Also, if we use the
pattern of exposing only the smart constructor (user), and keeping the
data constructor (User) hidden, then a User record can only be used in
contexts where all the invariants must always be held, which can be
inflexible.

### digestive-functors formlet style ###

**digestive-functors** solves the multiple validations problem. Our
formlet could look like:

```haskell
userForm :: (Monad m) => Form Text m User
userForm = User
  <$>  "firstName"    .: validate ((notEmpty >=> startsWith "A") >>> eitherToResult) (text Nothing)
  <*>  "lastName "    .: validate  (notEmpty >>> eitherToResult)                     (text Nothing)
  <*>  "emailAddress" .: validate  (notEmpty >>> eitherToResult)                     (text Nothing)
```

But, how do we handle the email confirmations? Since formlets are
applicatives and not monadic, one way we could do this is by threading
state through the monad and using validateM, but that's mistake prone,
and puts unnecessary constraints on our monad. We could intercept the
View record from **digestive-functors** once the form has been rendered,
but then we're splitting our validation logic into two different places.

ideas behind the structure of "validations"
-------------------------------------------

**validations** is based around 4 different data types. First, a
*Checker* is function with type

```haskell
a -> Either e b
```

Checkers tend to be non domain model specific, reusable pieces of code.
For example,

```haskell
nonEmpty :: (Monoid a, Eq a) => Checker Text a a
nonEmpty x =
  if (x == mempty)
     then Left "is empty"
     else Right x
```

. Notice that a Checker can transform its input as well, which consumer
is free to ignore. This is useful for turning string typed user input
into structured data.

A *Monadic Checker* is the same as a checker, but with type

```haskell
(Monad m) => a -> m (Either e b)
```

.

The next data type is a *Validator*. It's a function with type

```haskell
a -> monad (Either (errorKey, errorValue) b)
```

It's very similar to a Monadic Checker, but it also uses an "errorKey"
type. This allows us to map a validator failure back to some given input
(e.g. a form input field). If you look at the type signature for a
Validator, you'll notice that it's very similar to

```haskell
a -> m b
```

, so it's a [Kleisli
category](http://www.haskell.org/haskellwiki/Monad_laws), but the Either
inside doesn't allow us to use Haskell stuff for composing Kleisi
categories (e.g. `>=>`). However, we do know how to unwrap and rewrap
Either, so `validations` provides an instance of Category to allow for
validator composition. To create validators, we typically combine either
a checker with a field name using

```haskell
attach :: (Monad m) => Checker ev a b -> ek -> Validator ek ev m a b
```

or

```haskell
attachM :: (Monad m) => MonadicChecker ev m a b -> ek -> Validator ek ev m a b
```

for monadic checkers. Both `attach` and `attachM` are included with
this library, but you're free to wrap any conforming function (the
`Validator` data constructor is public.)

The final important data type is a *Validation*. The type of a
validations is:

    state -> monad (newState, errors)

where state is a type like a user record. newState is typically the same
type as state, but a transformation is allowed. Validations can be
constructed with

```haskell
validation :: (Monad m) => Lens b s -> a -> Validator ek ev m a b -> Validation [(ek,ev)] m s s
```

Validations also form a category and can be composed, similar to
Validators.

Example
-----------

Let's see the Validators and Validations in action. First, let's define
a Account record:

```haskell
data Account = Account
  { _name          :: Text
  , _accountNumber :: Text
  } deriving Show
```

Next, we want to define lenses for accessing and setting the fields. In
this example, we are using the internal lens functionality of
**validations**, but we'd typically use something like
[lens](http://hackage.haskell.org/package/lens) in our application.

```haskell
name :: Lens Text Account
name = lens _name (\s a -> s {_name = a})
```

```haskell
accountNumber :: Lens Text Account
accountNumber = lens _accountNumber (\s a -> s {_accountNumber = a})
```

We also want to use **digestive-functors** to define our form to bring
data in.

```haskell
nameField :: Text
nameField          = "name"

confirmNameField :: Text
confirmNameField   = "confirmName"

accountNumberField :: Text
accountNumberField = "accountNumber"

accountForm :: (Monad m) => Form Text m (Text, Text, Text)
accountForm = (,,)
  <$> nameField           .: (text Nothing)
  <*> confirmNameField    .: (text Nothing)
  <*> accountNumberField  .: (text Nothing)
```

We don't use field names directly in our formlet because we need to use
them in our validation as well. Also, notice that we are outputting a
3-tuple instead of a Account record. This is because there isn't a one
to one correspondence between our input fields and Account record fields
(the name confirm field will be discarded). So, our validation looks
like

```haskell
lengthIs :: Int -> Checker Text Text Text
lengthIs predicate txt = case (compareLength txt predicate) of
  EQ -> Right txt
  _  -> Left "account number not correct length"
```

```haskell
accountValidation :: (Monad m) => (Text, Text, Text) -> Validation [(Text, Text)] m Account Account
accountValidation (f1, f2, f3) = 
  validation name f1 (
    notEmpty        `attach` nameField
    >>>
    (f2 `confirms`) `attach` confirmNameField
  )
  >>>
  validation accountNumber f3 (
    notEmpty    `attach` accountNumberField
    >>>
    (lengthIs 10) `attach` accountNumberField
  ) 
```

What's going on here? First, since both `Validations` and `Validators` are
Categories, we can use the (`>>>`) operator from `Control.Arrow`. If
you're unfamiliar with this operator, for normal functions,

```haskell
a >>> b ≡ b . a
```

so you can think of it as a composition operator that reads left to
right.

Next, let's take a look at our first validation. It takes the "f1"
parameter passed into "accountValidation" and feeds it into "notEmpty".
If "notEmpty" returns a "Left", then the validation will make no state
changes, and instead adds

```haskell
("name" , "is empty")
```

to its "errors" value. On the other hand, if "f1" is not empty, it will
be passed onto the confirms function, where similar validation will
happen. If confirms also succeeds, the outputted value will passed to
the "name" lens, and the outputted state will be mutated with a new name
value.

testing
-------

Testing a validation is simple. For example,

```haskell
runIdentity $ (runValidation  (accountValidation ("hi", "hi", "1234567890")) Account { _name = "", _accountNumber = "" } :: Identity (Account, [(Text, Text)]))
```

yields

```haskell
(Account {_name = "hi", _accountNumber = "1234567890"},[])
```

.

```haskell
runIdentity $ (runValidation  (accountValidation ("hi", "bye", "12345678900")) Account { _name = "", _accountNumber = "" } :: Identity (Account, [(Text, Text)]))
```

yields

```haskell
(Account {_name = "", _accountNumber = ""},[("confirmName","fields do not match."),("accountNumber","account number not correct length")])
```

.

digestive-functors integration
------------------------------

Integration with **digestive-functors** is also pretty simple. Instead
of

```haskell
posted :: (Monad m) => m (View Text, Maybe (Text,Text,Text))
posted = postForm "f" accountForm $ testEnv [("f.name", "hello"), ("f.confirmName", "hello"), ("f.phoneNumber", "1(333)333-3333x3")]
```

add `validateView` in as well, like

```haskell
validatedPosted :: (Monad m) => m (View Text, Maybe Account)
validatedPosted =  posted >>= validateView accountValidation Account { _name = "", _accountNumber = "" }
```

You can also use `validateView'` if your domain record has a Monoid
instance.
