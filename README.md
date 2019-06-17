# sorbet-rails
Set of tools to make sorbet work with rails seamlessly.

This gem adds a few rake tasks to generate RBI for dynamic methods generated by Rails. It also includes sigs for related Rails classes. The generated rbis are added to `sorbet/rails-rbi/` folder.

Please feel free to send PR requests or file issues to improve the functionality of the gem!

## Initial setup

Following the steps below to setup the rbis after `srb tc`
1. Include ActiveRecord RBI
```
  $ srb tc sorbet-typed
```
2. Generate routes RBI
```
  $ rake rails_rbi:routes
```
3. Generate models RBI
```
  $ rake rails_rbi:models
```
4. Auto-upgrade the typecheck level of files
```
  $ srb tc --suggest-typed --typed=strict --error-white-list=7022 -a
```
Because we've generated RBI files for models & routes, a lot more files should be type-checkable now

## ActiveRecord RBI

There are ActiveRecord RBI that we vendor with this gem. Please run `srb tc sorbet-typed` to include the provided RBI in your `sorbet/rbi_list`

## Routes RBI
The following rake task generates `_path` and `_url` methods for all named routes defined in `routes.rb`
```
  rake rails_rbi:routes
```
## Models RBI
The following rake task generates rbi files for all models in the Rails App (all descendants of ActiveRecord::Base)
```
  rake rails_rbi:models
```
You can also regenerate rbi files for specific models
```
  rake rails_rbi:models[ModelName,AnotherOne,...]
```
At the moment, the generation task generate the following signatures
- Column getters & setters
- Associations getters & setters
- Enum values, checkers & scopes
- Named scopes
- Model relation class

## Trick & Tips
### `find`, `first` and `last` 
These 3 methods can either return a single nilable record or an array of records. Sorbet does not allow us to define multiple sigs for a function. It doesn't support defining one function sig that has varying returning value depending on the input param type. We opt to define the most commonly used sig for these methods, and monkey-patch new functions for the secondary use.

In short:
- Use `find`, `first` and `last` to fetch a single record
- Use `find_n`, `first_n`, `last_n` to fetch an array of records.

### `find_by_<attributes>`, `<attribute>_changed?`, etc.
Rails supports dynamic methods based on attribute names, such as `find_by_<attribute>`, `<attribute>_changed?`, etc. They all have static counterparts. Instead of generating all possible dynamic methods that Rails support, we recommend to use of the static version of these methods instead (also recommended by RuboCop). We've added sigs for th

Following are the list of attribute dynamic methods and their static counterpart. The static methods have proper sigs.
- `find_by_<attributes>` -> `find_by(<attributes>)`
- `find_by_<attributes>!` -> `find_by!(<attributes>)`
- `<attribute>_changed?` -> `attribute_changed?(<attribute>)`
- `saved_change_to_<attribute>?` -> `saved_change_to_attribute?(<attribute>)`

### `after_commit` and other callbacks
Consider codemod-ing `after_commit` callbacks to use instance method functions. Sorbet doesn't support binding an optional block with a different context. Because of this, when using a callback with a custom block, the block is evaluated in the wrong context (Class-level context). 

```
after_commit do ... end
```
codemod to 
```
after_commit :after_commit_func
def after_commit_func
  ...
end
```

See this link for a full list of callbacks available in Rails:
https://api.rubyonrails.org/classes/ActiveRecord/Callbacks.html

### Look for `# typed: ignore` files

Because sorbet initial setup tries to flag files at the typecheck level that generates 0 errors, there may be files in your repository that is `# typed: ignore`. This is because sometimes Rails allow very dynamic code that Sorbet does not regard as typecheck-able. 

It is worth going through the list of files that is ignored and resolve them (and auto upgrade the types of other files -- see Initial Setup #4). Usually this will resolve in many more files typecheckable.
