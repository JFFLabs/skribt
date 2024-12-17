# Skribt

Skribt is a simple system that is designed around configurations.  Configurations define the steps and flow through those steps to resolve a given support ticket.

When a support agent first enters the system, they can create a new incident (integrations to come later) and enter information about the support request.  The system will then use AI to try to summarize and match the support request to a most likely problem.  The agent selects the likely problem and begins down the resolution path.

Each problem, path, or sub-path the agent has to overcome is defined in its own configuration file.  Configuration files are `INI` format with a number of sections and sub-sections.  Each section contains one or more key names paired with a value.

```ini
[section]

	name = value

	[&.subsection]

		name = value
```

The values of the `INI` configuration can contain `JSON` for more advanced configuration.

```ini
[section]

	name = {
		"complex": "value",
		"example": true
	}
```

The key names and supported values carry specific meaniing.  As a matter of simplicity, Skribt maintains only a handful of valid key names and value structures to drive the resolution path.  Many of these keys are common across sections and sub-sections within the `INI` configuration file and carry the consistent behavior.  Key names are always 4 character words.

Let's take a look!

## Creating a Configuration

To create a configuration, create a file in the `problems` folder named after the actual problem a user is having.  Note, we do not mean _how_ the user is presenting the problem.  Someone might say "I can't log in."  In that case, the "problem" is not that they "cannot login" but it could be 1 of many problems.  They may have forgotten their password, or they may have forgotten their username, or there could be an issue with their internet connection.

Users don't always know what is actually going on.  It's up to the support agent to:

1. Figure out what's going on.
2. Once they know what's going on, figure out how to solve it.

Note the name of the example configurations below:

```
problems
  |- forgot-password.ini
  |- forgot-username.ini
```

Good practices:

- Keep configuration names short but descriptive.
- Keep configuration names all lowercase.
- Separate words with a dash.

Sticking to good practices will make these names easier to remember and reference as it's possible for configurations to import/use other configurations.

### Defining the Problem

The configuration file begins with the `[problem]` section and should minimally define the `text` and `info`:

```ini
[problem]

	text = The user has forgotten their password.
	info = (
		The user is unable to log into the system.  The most likely cause of this is that they have forgotten their password.  You need to assist the user in resetting their password.
	)
```

### Defining the Solution

Next, you need to define the `[solution]` section.  The solution will be paired with the problem so that the agent gets an idea of the overall process and what the outcome should look like.  The solution does not require separate `text` as it will always be presented in the context of the problem.  It does, however provide it's own `info` and additionally tells Skribt the **first** step to `goto` in order to solve the problem.

```ini
[solution]

	goto = step.selfReset
	info = (
		We will walk the user through an attempt at resetting their own password.  If that fails or the user is unable to reset their own password for whatever reason, we will attempt to manually reset it through the user management screens.
	)
```

### Defining the Resolution Path (Steps)

Every resolution path is comprised of one or more steps.  Each step needs to be addressed, receive some sort of response, and then move on to the next step based on that response.

To define a step, you will create a new section in the form of `[step.<name>]` where the `<name>` portion is defined by you.

- The `<name>` portion can contain letters A-Z and numbers, letters can be uppercase or lowercase
- The `<name>` must start with a letter.
- The `<name>` portion must be unique for each step.

Good Practices:

- Keep step names short and descriptive of what is happening in the step.
- Keep step names `lowerCamelCase` (first character lowercase, each new word starts uppercase)

Step names are frequently referenced (as we saw in the `[solution]` section's `goto` value).  By using `lowerCamelCase` we differentiate them from configuraiton names which should be in `lower-snake-case` (since both can be referenced in the configuration).

#### Define Our First Step (selfReset)

```ini
[step.selfReset]

	menu = Self Password Reset
	text = Direct the user to try using the reset password function of the website.
	info = (
		Direct the user to https://example.com/account/reset and have them follow the onscreen instructions which should tell them to enter their username, and click the "Request Reset Email."  The user should receive an email within a few seconds to a minute.

		If the user receives the email, they can follow the link provided in the email and enter and confirm their new password.  They will then be redirected to a login page where they can make sure the new password permits access.
	)
```

The only new key name we see here is the `menu` configuration.  This directs Skribt to add this step to the sidebar menu so that the agent can easily navigate back to that step to change the answer or start again from that position.

As with the `[problem]` and `[solution]` section, we see the `info` key being used to provide multi-line text describing in greater detail what the support agent should do or direct the user to do.

We also, again, see the use of the `text` key. The value of this field for a step is generally a 1 sentence summary describing what is being done, in many cases it may also be phrased as a question.  It is important to remember how this step was initially presented via the `text` configuration, though, as you'll want to make the possible agent responses (input) coherent.  For example we could make a far more complex configuration that begins by simply by asking the agent, "Has the user attempted to reset their password themselves?" with a "yes" or "no" input and then move on from there.

##### Defining Step Inputs

Once the step is defined, you need to define inputs for the agent.  Inputs are a new section in the INI file that begin with the following:

```ini
[&.input.<name>]
```

Similar to steps, you can define names as you wish but should conform to the same rules and good practices as with steps.  The `&` at the beginning indicates that this input is tied to the previously defined step.  It is important to make sure that all inputs related to a step fall between the starting `[step.<name>]` to which they belong, and before the next step's section opening.

Each step should generally have two or more inputs, as a step with only one input would indicate there's only one possible answer.  In our current example, the first input option will be very straightfoward:

```ini
[&.input.success]

	type = option
	text = The self-reset worked, user can now log in.
	exit = true
```

At this level, the `type` is always equal to `option` which indicates this is an option which the agent has to select as a possible outcome of the step.  All options are presented equally and are listed to the agent in the user interface in the order in which they're defined.  Simply move one input above another if you want it to appear earlier.

The `exit` key indicates that by choosing this option, the agent will exit the resolution path.  When the value is `true` it means that the resolution path was successful in doing whatever it set out to do.  When the value is `false` it means the resolution path was not successful.

The `text` value, again, is the basic summary which displays to the agent for that option.

As indicated, however, things are not always this simple, so let's take a look at more examples by defining another input.

```ini
[&.input.noEmail]

	type = option
	text = The user did not receive the reset email.
	goto = step.checkSpam
```

In the above example, we have now given the agent the ability to indicate that the user's self reset failed because they did not receive the reset email.  Critically, this differs from the previous input option because it uses the `goto` configuration key.  Remember that when we defined the `[solution]` we used the `goto` to indicate the first step to go to.  Here, we use it to define the next step to go to _if_ the agent selects this option.

In addition to defining multiple input options, we may also want the agent to log certain details about the interaction so that we can try to better identify issues.  Let's define another input that will be requested when the user selects the `&.input.noEmail` option.

```ini
[&.input.checkedEmail]

	when = noEmail
	type = email
	text = The email the user checked for the password reset
	data = userEmail
```

Here, we have added two new configuration keys.

The `when` key is used to define when this input appears.  By specifying `noEmail` we are telling Skribt that when the agent selects the option indicating the user did not receive the reset email, we should also prompt them to enter the email address of the user.

Unlike input options, we can also see that the `type` is now set to `email` indicating that the input is an email and should be validated as such.

Lastly, we have also added the `data` configuration key.  This key indicates that the data should be added to a general data pool with useful information about the entire incident.  If this key is set then two things happen:

1. If another input anywhere else in the resolution path (including sub-paths) shares the same `data` value, the agent will not be prompted again for that information.
2. This data can be easily viewed or modified from the data view of the incident.

###### Collecting Non-Option Inputs on Multiple Options

It is possible to collect the same input information for more than one option input.  For example, if we wanted to collect the user's email whether or not they received their password reset email, we could do:

```ini
		when = ["success", "noEmail"]
```

###### Available Non-Option Types

- `string` - A free-form string input that won't be validated
- `email` - A string input that will be validated as an email address
- `select` - A select dropdown allowing the agent to select a value

If the `type` is set to `select` you will also need to define a `list` configuration.  For example, if we also wanted the agent to enter how long the user waited for the email:

```ini
[&.input.emailWaitTime]

	when = ["success", "noEmail"]
	type = select
	text = Time waited for reset email
	list = [
		"1 minute or less",
		"1 to 2 minutes",
		"2 to 3 minutes",
		"3 minutes or more"
	]
```

#### Using Sub-Paths

Sometimes you want to break apart a resolution path into multiple distinct paths which could, stand on their own, but may also be useful.  Let's introduce a third option to our current step.  In this example, we will allow the agent to select a third input option.  In our previous two options, we assumed the user knew their username in order to request a password reset.  When we began with the example of our configuration file structure, however, we sneakly included two examples:

```
problems
  |- forgot-password.ini
  |- forgot-username.ini
```

In order to achieve this we'll need to define a couple new input options _and_ an new step.

```ini
[&.input.forgotUsername]

	type = option
	text = The user does not know their username
	goto = step.forgotUsername

[&.input.knowsEmail]

	when = forgotUsername
	type = select
	text = How certain is the user of their registered email?
	list = {
		"super": "Near totally certain",
		"kinda": "Not very certain"
	}
```

Now, we can define our `forgotUsername` step in the current `forgot-password.ini` configuration to use our `forgot-username.ini` configuration.  We do this by defining the `uses` configuration key, which basically means that this step is a proxy for simply jumping into that resolution path as a "sub process."

```ini
[step.forgotUsername]

	uses = forgot-username
	goto = {
		;
		; Jump to the `selfLookup` step in the `forgot-username.ini` resolution path
		; configuration, if:
		;
		; - We haven't gotten an answer on this yet
		; - ...And, the answer on the `input.knowsEmail` select is "super"
		;

		"step.forgotUsername.selfLookup": [
			[
				"!step.forgotUsername",
				"step.selfReset.input.knowsEmail.is(super)"
			]
		],

		;
		; Jump to the `systemLookup` step in the `forgot-username.ini` resolution path
		; configuration, if:
		;
		; - We haven't gotten an answer on this yet
		; - ...And, the answer on the `input.knowsEmail` select is "kinda"
		;

		"step.forgotUsername.systemLookup": [
			[
				"!step.forgotUsername",
				"step.selfReset.input.KnowsEmail.is(kinda)"
			]
		],

		;
		; Return to `selfRest` step in this resolution path, if the sub-path exited
		; with `true`.  Note: There is no additional goto below this one for the case of
		; the `forgotUsername` sub-path exiting with `false` because if the username
		; cannot be identified it can stop on an escalation step with no `goto` or `exit`
		; value.
		;

		"step.selfReset": [
			[
				"step.forgotUsername.?",
			]
		]
	}
```

When a step `uses` another resolution path, it doesn't need a `text` configuration because it simply passes off to a starting point in the used resolution path via the `goto` keyword, which means the next step that the agent will see is determined by the `goto` (as is the text, corresponding input options, etc).

At this point, you're probably asking us to slow down.  **What just happened?**  We went from pretty basic to super complex in one fell swoop!  Let's break down some new concepts one by one.

##### Mapped Lists

As noted previously, you can let the agent select from a drop down list of values when defining a non-option input as `type` equal to `select`.  However, our first example only provided an array of options, e.g.:

```json
[
	"Option 1",
	"Option 2",
	"Option 3"
]
```

In this most recent example we went from a JSON-style array to a JSON-object with defined keys.  Using the object syntax, the key is a known value that can be checked against.  The agent will still only see the text, such as "Near totally certain," in the drop down, but when selecting it, the value will be registered as "super".

We can then reference this value in functions, like we see later for conditional input checks e.g. `step.selfReset.input.knowsEmail.is(super)`

Good Practices:
- While it's possible for mapped list values to contain almost any character, to avoid having to escape certain characters when typing them, it's good to avoid characters that conflict with the condition syntax.  Almost certainly avoid periods, parenthesis, and quotes.
- Keep your mapped values short and somewhat expressive, ideally the same length to make the configuration more readable and consistent.
- Mapped values are always interpreted as strings, so while there are numeric-ish comparison functions like `gt()` for "greater than" these are still ultimately string comparisons, try not to rely on such if you don't know the natural sorting of characters and strings.

##### Conditional Goto

Review the previous step and the structure of the `goto` value.  While the comments in the code explain what each entry means, let's review some basic principles.  Take the following structure:

```ini
goto = {
	"step.<name>.<sub-name>": [
		[
			; conditions
		]
	],
	"step.<name>": [
		[
			; conditions
		]
	]
}
```

The key in the primary `goto` JSON object refers to the step which we are going to, however, we only go to that step if all the conditions are met.  Steps, when defined with the `uses` configuration key, require the name of the step to jump to in the included sub-path, `<sub-name>` in the above. Steps in the same configuration file only require the `<name>`.

Skribt will iterate over each of the keys in the `goto` object and test each set of conditions.  If all conditions are true, that is the next step.

###### Or Conditions

It is possible to say "go to this step if X (set of conditions) is true _or_ Y (set of conditions) is true."  To do this, you would define multiple sets of conditions.  For example:

```ini
goto = {
	"step.<name>": [
		[
			; If all conditions are true go to "step.<name>"
		],
		[
			; ... Or if all these conditions are true, also go to "step.<name>"
		]
	]
}
```

In short, adding additional sets of conditions create means that either set can match.  Within a given set, all conditions must be true, so it is effectively operating as "and."

###### Defining Conditions

Conditions generally come in only a handful of forms.  To check a form input you can add a conditions such as:

```ini
"step.<name>.input.<input-name>"
```

If ther is any "truthy" (a non-empty string, a numeric equivalent > 0, etc) value in that input, the condition is met.

If you're storing input values into the common data set via the `data` configuration key, you can check them as follows:

```ini
"data.<name>"
```

To invert a condition, i.e. to check if it's "not true" you can preceed it with a `!`:

```ini
"!data.<name>"
```

You can check whether or not a step has been "visited" by simply referring to the step:

```ini
"step.<name>
```

This will return a truthy value if the step has been visited.  Or a falsey value if it has not.  Additionally, if the step makes use of the `uses` configuration to refer to a sub-path, you can check the exit code from that sub-process as follows:

```ini
"step.<name>.?
```

Lastly, for any input, you can use several conditional functions to check for more specific values.  For example, to see that an input value is equal to a specific value, you can use the `is()` function:

```ini
"step.<name>.input.<input-name>.is(<value>)"
```

Additional functions include the following:

- `gt(<value>)`  - The value inputted is greater than `<value>`
- `lt(<value>)`  - The value inputted is less than `<value>`
- `has(<value>)` - The value contains the `<value>`

To check that an input value is less than _or_ equal to another value, you would invert the greater than condition using the prefixed `!`, e.g.:

```ini
"!step.<name>.input.<input-name>.gt(5)
```

Not greater than five means less than or equal to five.



