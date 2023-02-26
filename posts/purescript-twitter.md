---
categories:
- markdown
date: '2020-04-01'
description: Learn how to call a Twitter API using PureScript
layout: post
title: "Introduction to PureScript: Twitter Search API"
toc: true

---

*This post is an ported, edited version of the [original](https://medium.com/@rajiv.abraham/introduction-purescript-twitter-fec6df5276dc)*

*TLDR*: I wrote this in a fiction format for fun. The actual code is in the [repo](https://github.com/RAbraham/blog-twitter-purescript-simple). Also, I'm new to FP so this is newbie code. I can refactor it to be elegant but I want to keep this simple for beginners.


## Twitter Storm
Kim Kardashian felt uneasy as soon as she woke up. She had just used the r-word yesterday and suffered a huge [backlash](https://www.cosmopolitan.com/entertainment/a24515906/kim-kardashian-r-word-halloween-costume/). She felt vulnerable about her twitter following and needed to be reassured. She had to do something different. Yes, she could just type her name in the Twitter App and see what people were saying about her. But she had secretaries for that. No, she had to do what no other celeb had done before. She would code!

What language though? A language which is nice and clean and pure. So she googles around and discovers PureScript! She installs it in a breeze while wondering about this Mr. Java Script guy who was always complaining online on how difficult it was. Sigh. Ok, what next?

## Reading Twitter credentials

First, she has to read her Twitter credentials from a file. Yes, she could hard code the passwords in the program but she's a celeb. She knows _Security_.

So, she got her credentials from Twitter and created a file like below at `config/twitter_credentials.json`
```
{
  "consumer_key": "KimMama",
  "consumer_secret": "KimLikesToCode",
  "access_token": "KimDoesNotKnowWhatThisIsFor",
  "access_token_secret": "KimThinksTwitterHasGoneMad"
}

```
She built a JavaScript like object in PureScript(called records) using `type`:
```
type TwitterCredentials =
  { consumer_key :: String
  , consumer_secret :: String
  , access_token :: String
  , access_token_secret :: String
  }

```

How do we read this file?

```haskell
import Node.Encoding (Encoding(..))
import Node.FS.Aff (readTextFile)

readConfigStr :: String -> Aff String
readConfigStr path =  readTextFile UTF8 path

```

`import Node.Encoding (Encoding(..))` meant import the type constructor `Encoding` and  the `..` meant import all it's data constructors as well, one of which is `UTF8`. Since she is a celeb and she is never wrong, type constructors are like abstract base types and data constructors are like normal OOP constructors but fancier. You can have data constructors with different names and you can even treat them like Enumerations in switch/case like statements(Kim's BFF liked to call them pattern matching).

`Aff` stands Asynchronous Effect(the synchronous effect is called `Effect`). These effects _represent_ an action that the program would like to take, but _not executed_ yet. Whaaa?

If Kim wanted to call Khloe for lunch, buy flowers for her mother and type her next tweet... She wouldn't be the person doing it, would she? It would be her _secretary_! All, she would do is `text` her secretary commands to do this thing but it wouldn't happen until her secretary *actually executed* the commands at a later time!

In the same way, `Aff`(and `Effect`) were like `texts` by Kim to her new secretary `PureScript`. It was a way of telling `PureScript` that she wanted them to be done but it was just a _representation_ of a command, not the actual _execution_  of the command. By representation, it just meant it was a value, just like the way number `3` or `"a_string"` or a JavaScript object were values.

For e.g., imagine the following pseudocode in an imperative language(e.g. Python):

```
1: x = print("A String")
2: x
3: x

```
The output would be
```sh
A string
```
The `execution` and `evaluation` of the `print` statement both happen at line 1.

But in a functional language, the above pseudocode would be something like
```
1: run(
2: let x = print("A String")
3: x
4: x
5: )
```
And the output would be
```sh
A String
A String
```
`let` is like the variable assignment in imperative code.

Only the evaluation happens at lines 2-4 but *not the execution*. The execution happens inside `run`. So before the program is given to run, `x` replaced at lines 3 and 4 to be `print("A String")`. Note, the `print` has different interpretations. In the imperative setting, it executes a command, but in the functional setting, it executes nothing, just returns back a value representing an action for future execution by the `run` procedure.

Another viewpoint is that most applications always start with the `main` function. In PureScript, perhaps the simplest program one could write is.

```
import Effect.Console (log)
main:: Effect Unit
main = log "Product Placement Here. ;)"
```
The signature for `log` is `log :: String -> Effect Unit`. `Unit` stands for nothing, as in, we don't expect anything back from the console.

And like the pseudocode above, what happens within PureScript code, unseen by the programmer is something like

```
run(main)
```

Kim felt a chill through her spine. She regretted not taking programming seriously in school.


Ok, `readConfigStr` returned a `Aff String` but she needed to convert it to our `TwitterCredentials` record. She asked her secretary for technology to find a library for her and she found [PureScript-Simple-JSON](https://purescript-simple-json.readthedocs.io/en/latest/) by a guy called [Justin Woo](https://github.com/justinwoo).

```
import Simple.JSON as SimpleJSON
import Data.Either (Either(..))

parseConfig :: String -> Either String TwitterCredentials
parseConfig s =
  case SimpleJSON.readJSON s of
    Left error -> Left (show error)
    Right (creds :: TwitterCredentials) -> Right creds
```

`parseConfig` has an `Either String TwitterCredentials` in it's signature. It's like an union type. The result could either be a String(an error string) or the actual credentials. PureScript defines `Either` as

```
data Either a b = Left a | Right b
```

So if we want to return a string, we return `Left "my error string"`, the actual credentials as `Right creds`. That way, the person calling `parseConfig` knows which is which.

In `parseConfig`, `SimpleJSON.readJSON` returned an `Either` but Kim didn't want to deal with the complex `Left` type, so she just converted that to a string using `show`.



Now it was just a matter of calling `readConfigStr` and passing the value to `parseConfig`. Something like this pseudocode

```
cStr = readConfigStr path
return parseConfig cStr
```
But she couldn't make it compile! She started panicking and thought of what would happen if the word got out and Taylor Swift found out. _The Shame_

"Try the _do_ notation", said a voice from behind.

Kim swivelled back and her mouth opened with surprise.

"_Kanye_! I didn't know you knew PureScript!"

"Nah, PureScript is for hipsters. I'm old school. I like my Haskell."

He continued, "The _do_ notation allows you to extract the `String` from `Aff String` and gives you the illusion of the pseudocode above."


```
readConfig :: String -> Aff (Either String TwitterCredentials)
readConfig path = do
  cStr <- readConfigStr path
  pure $ parseConfig cStr
```

"What's `pure $` for?", asked Kim?

Kanye sighed. He knew the author of this post was in a hurry to move on to doing [cooler stuff](https://blog.rajivabraham.com/posts/purescript-serverless) and didn't want to get into monads in this post. So he bailed too.

First `$`. That's just a simple way of saying consider everything after as one value. For e.g.
`show $ SimpleJSON.readJSON s` meant `show (SimpleJSON.readJSON s)` instead of `(show SimpleJSON.readJSON) s`. Kim approved. She liked `$` signs.

Kanye then braced himself for his 'simplification' of `pure`.

"You noticed that it was `cStr <- readConfigStr path` and not `let cStr = readConfigStr path`. The `<-` is syntax sugar which make it look like an `=`. But what is really happening underneath is something very similar to callbacks. The `Aff String` type has to be given a function to work on the `String` value within it. But this function can't just be `cStr -> parseConfig cStr`. The function has to return back an `Aff something`. `pure` is a constructor. In this context of `Aff`, when we say `pure something`, it's like saying `new Aff(something)` or in our case, it's like saying `new Aff(parseConfig(cStr))`"

Kim beamed at Kanye. He looked so hot right now. She wanted him so bad.

## Bearer Token from Twitter.

Great, that gave her the credentials but she needed a bearer token from Twitter which she would then use to get the results. How does one call the Twitter endpoint in PureScript? She beckoned her secretary for technology to find her a library. Her secretary came back running.

"I found a library called [Milkis](https://github.com/justinwoo/purescript-milkis)... _again by Justin Woo_!"

Kim's eyes sharpened with intent. She wondered out aloud, "Do you think this Justin guy is a celebrity in the PureScript world? Hmmmm _make my agent call his agent. Let's do a reality show together._"



Kim first created a method to construct the authorization string from the credentials and encode it in `Base64`. The `<>` was like an append operator.

```
import Data.String.Base64 as S
authorizationStr :: TwitterCredentials -> String
authorizationStr credentials =
  S.encode $ credentials.consumer_key <> ":" <> credentials.consumer_secret

```

She then made a simple `fetch` helper method from `Milkis`.

```
import Milkis as M
import Milkis.Impl.Node (nodeFetch)


fetch :: M.Fetch
fetch = M.fetch nodeFetch
```


She then created a method to get the bearer token string or return a string as error(in the `Left` part of the code).

```
import Milkis as M
import Effect.Aff (Aff, attempt)

getTokenCredentialsStr :: String -> Aff (Either String String)
getTokenCredentialsStr basicAuthorizationStr = do
    let
      opts =
        { body: "grant_type=client_credentials"
        , method: M.postMethod
        , headers: M.makeHeaders { "Authorization": basicAuthorizationStr
                                 , "Content-Type": "application/x-www-form-urlencoded;charset=UTF-8"
                                 }

        }
    _response <- attempt $ fetch (M.URL "https://api.twitter.com/oauth2/token") opts
    case _response of
      Left e -> do
        pure (Left $ show e)
      Right response -> do
        theText <- M.text response
        pure (Right theText)

```

Now to bring it all together.

```

type BearerAuthorization =
  { token_type :: String
  , access_token :: String
  }

basicHeader :: String -> String
basicHeader base64EncodedStr = "Basic " <> base64EncodedStr

toBearerAuthorization :: String -> Either String BearerAuthorization
toBearerAuthorization tokenString = do
  case SimpleJSON.readJSON tokenString of
    Left e -> do
      Left $ show e
    Right (result :: BearerAuthorization) -> do
      Right result

getTokenCredentials :: TwitterCredentials -> Aff (Either String BearerAuthorization)
getTokenCredentials credentials = do
  tokenCredentialsStrE <- getTokenCredentialsStr $ basicHeader $ authorizationStr credentials
  case tokenCredentialsStrE of
    Left error -> do
      pure (Left error)
    Right tokenCredentialsStr -> do
      let tokenCredentialsE = toBearerAuthorization(tokenCredentialsStr)
      case tokenCredentialsE of
        Left error -> do
          pure (Left error)
        Right authResult -> do
          pure (Right authResult)
```

Great, we had the bearer token. It's finally time to search for `Kim Kardashian`!
PureScript had this interesting signature format though. What it was saying below was that `showResults` took as input a `BearerAuthorization` and a `String` and returned an `Aff (Either String SearchResults)`

Also, the `SearchResults` and `Status` had lots of fields but she just wanted the basic stuff.

```

type Status =
  { created_at :: String
  , id_str :: String
  , text :: String
  }

type SearchResults =
  { statuses :: Array Status
  }

twitterURL :: String -> M.URL
twitterURL singleSearchTerm = M.URL $ "https://api.twitter.com/1.1/search/tweets.json?q=" <> singleSearchTerm

showResults :: BearerAuthorization -> String -> Aff (Either String SearchResults)
showResults credentials singleSearchTerm = do
  let
    opts =
      { method: M.getMethod
      , headers: M.makeHeaders { "Authorization": "Bearer " <> credentials.access_token}

      }
  _response <- attempt $ fetch (twitterURL singleSearchTerm) opts
  case _response of
    Left e -> do
      pure (Left $ show e)
    Right response -> do
      stuff <- M.text response
      let aJson = SimpleJSON.readJSON stuff
      case  aJson of
        Left e -> do
          pure $ Left $ show e
        Right (result :: SearchResults) -> do
          pure (Right result)
```

Finally, reaching the very end to the `main` command!

```
import Effect.Class.Console (errorShow, log)
import Effect.Aff (Aff, launchAff_)

main :: Effect Unit
main = launchAff_ do
  let searchTerm = "Kim Kardashian"
  config <- readConfig "./config/twitter_credentials.json"
  case config of
    Left errorStr -> errorShow errorStr
    Right credentials -> do
      tokenCredentialsE <- getTokenCredentials credentials
      case tokenCredentialsE of
        Left error ->
          errorShow error
        Right tokenCredentials -> do
          resultsE <- showResults tokenCredentials searchTerm
          case resultsE of
            Left error ->
              errorShow error
            Right result ->
              log $ show $ "Response:" <> (show result.statuses)

```

`launchAff_` was required because the entire computation returned `Aff something` but `main` was of type `Effect Unit`. So `launchAff_` just converted `Aff something` to `Effect Unit`


As Kim beamed with pride at her code, she flashed her eyes at Kanye and asked him, "Isn't the code beautiful?"

Kanye gazed into her eyes and said, "Actually, it sucks. There are so many case statements in that code that I feel cross eyed."

And the next thing Kanye knew, was that he was flat on the ground, his jaw felt like it had been displaced and he was seeing double.

For there are three things you don't tell your wife:

1) Honey, you have gained weight
2) Your code sucks
3) I miss my mother's cooking.


As Kanye massaged his jaw, he muttered, ".. I guess she does not want to know about the `ExceptT` Monad.."

