# C5 Kohana Form Validation

This is a port of Kohana 2.3.4's form validation helper library. Its usage is vaguely similar to C5's built-in "validation/form" helper, but is much more full-featured in terms of the number of validation rules available.

## How To Use
Copy the kohana_validation.php file into your site's top-level `libraries` directory. Then, in a controller action that responds to a form submission, add some code like this:

	if ($this->post()) {
	    Loader::library('kohana_validation');
		$v = new KohanaValidation($this->post());
	
		$v->add_rule('name', 'required', 'Name is required.');
		$v->add_rule('name', 'length[0,255]', 'Name cannot exceed 255 characters in length.');
		
		$v->add_rule('email', 'required', 'Email is required.');
		$v->add_rule('email', 'email', 'Email is not a valid format.');
		
		$v->add_rule('username', 'required', 'Username is required.');
		$v->add_rule('username', 'alpha_dash', 'Username can only contain letters, numbers, underscores and dashes.');
		
		$v->add_rule('password', 'required', 'Password is required.');
		$v->add_rule('password', 'matches[password_confirm]', 'Passwords do not match.'); //assumes there is another field in the form named "password_confirm"
		
		$v->add_rule('image_fID', 'required', 'You must choose an Image.');
		$v->add_rule('image_fID', 'atleast[1]', 'You must choose an Image.'); //0 is posted by the asset library helper when there's no selection, so if the field is required you must explicitly check that the value is greater than 0
		//Note about file uploads: this library contains file upload validators, but you probably shouldn't use plain old "file" inputs in your form -- instead use C5's asset library helper, which posts a file id
		
		$v->pre_filter('trim', 'name'); //run a filter on the 'name' field prior to validation (php's built-in 'trim()' function in this case, but you could also create your own filter function that accepts a string and returns a string, similar to how the custom validation rules work)
		                                //(NOTE that param order is reversed from add_rule() -- field name is 2nd!)
		
		$v->validate();
		$error = $v->errors(true); //passing in true to get a C5 "validation/error" object (instead of an array)
	}

	if ($error->has()) {
		$this->set('error', $error); //If you're in the dashboard, C5 will automagically display these for you in the view
	} else {
		//Success!
		save_your_data($this->post()); //do whatever you need to do here -- add pages, save to database, send an email, whatever
		$path = $this->getCollectionObject()->getCollectionPath();
		$this->redirect($path, 'success_action');
	}
	
	//Note that if you use the Concrete5 form helpers to output your form fields, you do not need to worry about repopulating posted values -- the helpers will do that for you (using values from $_POST).

For a full list of available rules, see http://docs.kohanaphp.com/libraries/validation#rules

## Changes From Original Library
Note that I made a few additions and modifications to Kohana's library:

 * Added "inrange", "atleast" and "atmost" validation rules for validating that numbers are within a certain range, or greater than / less than a certain number.
 * Error messages are set in the library directly (not via a separate language file).
 * Added new `add_rule()` and `add_callback()` methods for adding one rule/callback at a time along with its error message. This makes the code easier to understand and maintain because the error messages are right there next to the applicable rules instead of in a separate data structure.
 * The `errors()` method will optionally return a Concrete5 `validation/error` object instead of an associative array if you pass in `true`.

## Custom Rules
If you need to validate something that isn't covered by the built-in rules, you can write your own custom rules. To do so, create a function that accepts one argument (the posted value), and returns true (if valid) or false (if invalid). 

_Note that the custom validation rule function you define MUST be public (because it is called from the validation helper library, not from within this controller). Be careful not to give it a name that is likely to be confused with an action (C5 unfortunately provides no way to define a public method in a controller that isn't an action responder -- you can have methods that aren't intended to be actions, but if someone accidentally hits that url, it will get called and confusion will ensue)._

Here is an example of a custom rule that checks if a username is available. First, assign the rule to a field in the validation object:

	$v->add_rule('username', array($this, 'validate_unique_username'), 'Sorry, that username is already taken');

Now create your custom rule function. This function must accept exactly one parameter -- the value that gets POSTed to the field. If the value is valid, return true. If the value is invalid, return false. For example:

   	public function validate_unique_username($value) {
		$username_exists = check_if_username_exists_in_database($value); //made-up function for example purposes
		if ($username_exists) {
			return false; //invalid (because it already exists in the database)
		} else {
			return true; //valid!
		}
	}

## Callbacks
For custom validation rules where you need to compare more than one field, you can add a callback. These are very similar to Custom Rules, except that with a callback you are passed the entire array of submitted data instead of just the one field's value. For example, let's say we want to validate that an end date is after a start date. First, add the callback to the validation object (around the same place you'd call `add_rule`):

    $v->add_callback('end_date', array($this, 'validate_end_date'), 'The end data must be after the start date.');

Now create your function that will get called back by the validation library. This function must accept exactly two parameters -- a "KohanaValidation" object (which you can treat as if it were an array to retrieve data values), and the field name that is being validated. If there is an error, call the `add_error` method of the KohanaValidation object (don't just return false). For example:

    public function validate_end_date(KohanaValidation $v, $field_name) {
		if (strtotime($v['end_date']) < strtotime($v['start_date'])) {
			$v->add_error($field_name, 'validate_end_date'); //2nd arg MUST match the function name
		}
	}

_Note: even though callbacks can inspect and add errors to any fields they want, you still must associate it with one field when calling add_callback() on the validation object. I'm not sure why this is, but it doesn't work otherwise._
