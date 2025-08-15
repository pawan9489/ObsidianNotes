* `if let` vs `match` - 
	* A `match` is exhaustive; you **must** handle every possible case.
	* `if let` is for when you only care about **one** specific pattern and want to do something different for all other cases. It's syntactic sugar for a match that only has two arms.
*  Use `match` or `if let` for errors you expect to happen (like user input). Use `.expect()` for errors that "should never happen" and indicate a bug in your code. Avoid `.unwrap()` on `Option` & `Result`.