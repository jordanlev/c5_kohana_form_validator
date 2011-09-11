# C5 Kohana Form Validation

This is a port of Kohana 2.3.4's form validation helper library. Its usage is vaguely similar to C5's built-in "validation/form" helper, but is much more full-featured in terms of the number of validation rules available.

# How To Use
Copy the kohana_validation.php file into your site's top-level `libraries` directory. Then, in a controller action that responds to a form submission, add some code like this:

	if ($this->post()) {
	    Loader::library('kohana_validation');
		$post = new KohanaValidation($this->post());
	
		$post->add_rule('name', 'required', 'Name is required.');
		$post->add_rule('name', 'length[0,255]', 'Name cannot exceed 255 characters in length.');
		
		$post->add_rule('email', 'required', 'Email is required.');
		$post->add_rule('email', 'email', 'Email is not a valid format.');
		
		$post->add_rule('username', 'required', 'Username is required.');
		$post->add_rule('username', 'alpha_dash', 'Username can only contain letters, numbers, underscores and dashes.');
		$post->add_rule('username', array($this, 'validate_unique_username'), 'Sorry, that username is already taken'); //This is a custom rule -- you're telling the helper to call a function you've defined in this controller class called "validate_unique_username", which will accept one argument (the posted value), and return true (if valid) or false (if invalid). Note that the custom validation rule function you define MUST be public (because it is called from the validation helper library, not from within this controller). Be careful not to give it a name that is likely to be confused with an action (C5 unfortunately provides no way to define a public method in a controller that isn't an action responder -- you can have methods that aren't intended to be actions, but if someone accidentally hits that url, it will get called and confusion will ensue).
		
		$post->add_rule('password', 'required', 'Password is required.');
		$post->add_rule('password', 'matches[password_confirm]'); //assumes there's another field in the form named "password_confirm"
		
		//Note about file uploads: this library contains file upload validators, but you probably shouldn't use plain old "file" inputs in your form -- instead use C5's asset library helper, which posts a file id
		$post->add_rule('image_fID', 'required', 'You must choose an Image.');
		$post->add_rule('image_fID', 'atleast[1]', 'You must choose an Image.'); //0 is posted by the asset library helper when there's no selection, so if the field is required you must explicitly check that the value is greater than 0
		
		$post->validate();
		$error = $post->errors(true); //pass true to get a C5 "validation/error" object instead of an array
	}

	if ($error->has()) {
		$this->set('error', $error); //If you're in the dashboard, C5 will automagically display these for us in the view
	} else {
		//Success!
		save_your_data($this->post()); //do whatever you need to do here -- add pages, save to database, send an email, whatever
		$path = $this->getCollectionObject()->getCollectionPath();
		$this->redirect($path, 'success_action');
	}
	
	//Note that if you use the Concrete5 form helpers to output your form fields, you do not need to worry about repopulating posted values in the even of validation failure -- those form helpers will automatically put in values it finds in $_POST!

For a full list of available rules, see http://docs.kohanaphp.com/libraries/validation#rules

Note that I made a few additions and modifications to Kohana's library:

 * Added "inrange", "atleast" and "atmost" validation rules (for validating that numbers are within a certain range, or greater than / less than a certain number).
 * Error messages are set in the library directly (not via a separate "language" file)
 * Added a new `add_rule()` method for adding one rule at a time along with its error message. This can make your code easier to understand and maintain since the error messages are right there next to the applicable rule (instead of in a separate data structure).
 * The `errors()` method will optionally return a Concrete5 `validation/error` object (instead of an associative array) if you pass in `true`.
