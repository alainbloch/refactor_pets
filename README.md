# README

Instead of this being a document to help you get the application started, this will detail on the interview testing criteria for this rails test.

This test covers many areas:
* Rails/Ruby familiarity
* Ability to navigate an existing app
* Debugging
* Refactoring
* Taking ownership
* Data structures and relations
* Data migrations
* Routing
* Good comprehension of OO patterns
* Understanding of best practices
* ..and more

## Rating
A good engineer should be able to get through most of the problems in a timely manner while demonstrating the areas listed above. The solutions should only be seen as guidelines to evaluating the answer and not taken strictly. It is not necessarily how many problems they solve but the quality of the solutions. Sometimes a quality solution can take more time (especially if they start testing).

As the interviewer, you should take care not to give any additional technical instruction or insight apart from what is needed to determine the interviewee's skill and comprehension. You can always give insight after solution has been achieved or if the candidate is stuck on the problem.

There is one particular area that I feel that can easily determine if the candidate is good: __does the candidate create and run tests?!__ Most of the initial problems can be discovered if the candidate starts first by running tests. A good analogy to a candidate running tests is [the golden snitch in quidditch](http://harrypotter.wikia.com/wiki/Golden_Snitch) - it earns the candidate a lot of merit.

### Synopsis
The interviewee is given an application that was abandoned by the previous developer. The client is desperate and doesn't know anything technically. The application allows for users to list their cats.

Listed below is all the problems that the interviewees should be given in order. Each solution will slightly show a better understanding of the test areas.

---

### Problem 1
_There is an error when trying to display the new cat form!_

####Solution
The cat form is trying to display _desc_ instead of _description_. The interviewer should simply change the form to use _description_ instead of _desc_.

---

### Problem 2
_There is an error when creating new cats!_

####Solution
The cat form is using `owner_id` instead of `user_id` column shown in the schema. There are two solutions to this and one is preferred over the other.

* __Average:__ Changing all instances of `owner_id` to be `user_id`.
* __Best:__ Migrate the database to use `owner_id` instead of `user_id` and change the Cat class to have `belongs_to :owner, class_name: 'User'`
Why is this the best way to do it? Because the rest of the app already is using `owner_id` and __owner__ is a more descriptive relationship between the user and the cat classes.

---

### Problem 3
_The user is not being associated with the cat!_

####Solution
Principally the current user should be assigned to the cat but there are several solutions to this problem and there is a specific solution that you should look for.

* __Worst:__ The hidden field in the cat form sets the user_id on the cat. This introduces a vulnerability in where a malicious user could assign a cat to someone else.
* __Average:__ The user is associated in the cat controller's create action. This can be done using either a merging of the current user id into the cat params like so `@cat = Cat.new(cat_params.merge({user_id: current_user.id}))`
or something similar.
* __Good:__ The user model is changed so that it `has_many :cats` instead of `has_one :cat` and then uses `@cat = current_user.cats.build(cat_params)` or something similar.
* __Best:__ Doing the above but also seeing that `owner_id` should be removed from `params.require(:cat).permit(:name, :description, :owner_id)` in the __cat_params__ method for security reasons.

---

### Problem 4
_A cat can be created without a name or description!_

####Solution
Validation needs to be added to the cat model.

* __Worst:__ Multi-line validations or hand-rolled validations:
``` ruby
validates_presence_of :description
validates_presence_of :owner
validate :needs_desc_and_owner

def needs_desc_and_owner
  ...
end
```
* __Average:__ Using the old way to do validation: `validates_presence_of :description, :owner, :name`
* __Good:__  Using the new way to do validation: `validate :description, :owner, :name, presence: true`

---

### Problem 5
_Right now the we are just displaying a cat's owner id and not the owner name! Let's display the owner's name instead._

#### Solution
__THERE'S A CATCH!__ Even though we submitted a name in the sign up form current user doesn't have a name! Why? Because the authentication gem Devise needs to be configured correctly to allow for the name to be submitted. So this is a two part answer. The first part is testing how the interviewee can easily debug and find the answers (Google, Stackoverflow). The second part is how we are going to display the first name.

####Solution: Part 1
On the [devise gem github page](https://github.com/plataformatec/devise), halfway down the README, there is a simple answer for allowing the name to be passed in:

```ruby
class ApplicationController < ActionController::Base
  before_action :configure_permitted_parameters, if: :devise_controller?

  protected

  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:sign_up, keys: [:name])
  end
end
```

####Solution: Part 2
The next step is to display the owner's name. There are a couple answers.

* __Average:__ calling `cat.owner.name` instead of `cat.owner.id`
* __Best:__ Doing the above but also in the cats controller index action, revising the `@cats` variable declaration to be `Cats.includes(:owner).all` so that we don't have N+1 queries.

---

### Problem 6
_Any user can edit or delete a cat! We should only allow the owners of cat to be able to edit or delete a cat record._

####Solution
We need to ensure that the current user is the owner of the cat before allowing them to perform any action or see any link to edit or destroy a cat.

* __Worst:__ Doing any inline logic within the view or in the controller actions to check that the current user is the cat owner: `if cat.owner_id == current_user.id`
* __Average:__ Creating two separate methods within the cats helper and in the cats controller that performs similar logic as above:
```ruby
  def is_owner?(cat)
    cat.user_id == current_user.id
  end
```
* __Good:__ Making the `is_owner?` controller method available in views by using `helper_methods`.
* __Good:__ Moving similar logic into the Cat model so that it can be reused wherever a cat instance exists:
```ruby
  class Cat < ActiveRecord::Base
  ...
  def has_owner?(user)
    user.id == self.id
  end
```
* __Best:__ Getting an authorization library like [cancancan](https://github.com/CanCanCommunity/cancancan) up and working in the app.

---

### Problem 7
_We need to have more than one type of pet!_ The client has discovered that the users have different types of pets that they want to add! We need to change the app so that we can support different types of pets.

####Solution
The solution to this problem is varied just like the others.
* __Worst:__ Doing anything that requires dropping the database or changing the cats table.
* __Worst:__ Creating a new table for each type of pet.
* __Average:__ Creating a new Pet model and table that will be used in deference to the Cat model. The Pet model should have a `type` column that will store the type of pet. Change all instances of the Cat model to use the Pet model. Have the different pet types contained in a constant located in the pet model. Adding a drop down on the pets form so that a user can select which kind of pet they are adding.
* __Good:__ Doing the above and then migrating the data from Cats table into the new Pet table, and deprecating the old cats table.
* __Good:__ Realizing that Cat model could possibly use STI and inherit from the Pet model so that in the future we can have different logic around each pet type.
* __Best:__ Doing a two part migration. First, migrating the data by using a reusable code block contained in a class, module or something similar instead of having the logic contained in the rails migration. Secondly, after checking that the data has successfully migrated, deprecating the old table and removing the old controller logic in another commit.

---

### Problem 8
_There needs to be multiple owners!_ The client has been told that a pet could have multiple owners! We need to allow a pet to have more than one owner and change the interface to allow for that to happen!

####Solution
* __Average:__ Use the HABTM rails method to create a relationship between the pets and the owners.
* __Good:__ Create a join model and use the `has_many .. through: ..` method since the join model can now be used to have other interesting bits of meta data such as when the person is added and who the primary owner might be.
* __Best:__ Doing something similar to the above two solutions and then adding an autocomplete dropdown on the pet form that allows for multiple owners to be added to the pet. Then changing so that all the owner names appear on the index and show pages.


---

### Problem 9
_We need owners to be able to upload multiple images for their pets!_

####Solution
* __Worst:__ a file input is added to the form, and the file is manually associated or marshalled with a hand-rolled method.
* __Good:__ Using a plugin like [paperclip](), [carrierwave]() or something similar.

---

### More Problems?!
Did the candidate solve everything? Wow! That's a lot. Here are some other bonus questions and problems that you can give them:
* How would you refactor this app?
* Styling the website better (twitter bootstrap)
* Internationalizing the site
* Making static pages (thoughtbot highvoltage)
* Creating admin roles
* Owner profiles
* Search functionality
* Filtering based on pet type
* Creating friendships between users
* Messaging
* Forums
* Contact page

