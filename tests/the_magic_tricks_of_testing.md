# The magic tricks of testing
# Talking about unit test

__Types__
* Query: Return something / Change nothing
* Command: Return nothing / Change something

__Messages for object testing__
* Origin incoming
* Sent to self
* Outgoing

__Rules__
* Test incoming query messages by making _assertions_ about _what they send back_
* Test incoming command messages by making _assertions_ about _direct public side effects_
* DON'T test private methods. Don't make assertions about their result. Do not expect to send them

__Concepts__
* We test the interface not the implementation
* We don't test private method, a test for private method make you can't improve the code without breaking the test, this doesn't make it safe to refactor, it makes impossible to do so!
* Simplicity is the heart of complexity
