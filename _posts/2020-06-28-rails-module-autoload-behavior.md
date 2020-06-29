---
layout: post
title: A strange behavior of Rails constant autoloading
author: Jinsik Park
author-email: jinsik.park@cre.ma
excerpt: Story about a peculiar error message we faced during development and how we attempted to fix it.
publish: true

---

### `A copy of UserCore has been removed from the module tree but is still active!`

- Error was emitted from method `ActiveSupport::Dependencies.load_missing_constant`
  - Method was called when a constant is missing inside some `Module` context. In this case: `UserCore`



### Necessary code context

```ruby
# app/models/user.rb
class User < ApplicationRecord
  include UserCore
end

# app/models/concerns/user_core.rb
module UserCore
  extend ActiveSupport::Concern

  def admin?
    role?(UserRole::ADMIN)
  end
end

# app/enums/user_role.rb
module UserRole
  include Enum

  UserRole.define :NONMEMBER, 5
  UserRole.define :USER, 10
  ...
  UserRole.define :ADMIN, 50
end
```



### Steps to reproduce
- In your rails console, enter:
  1. `u = User.first; u.admin?`
  2. `reload!; u.admin?`



### Why does this happen?
- When `User#admin?` is called, lookup for `UserRole` constant happens. When it is found, it is cached in the module constant `UserCore`: i.e. return value of `UserCore.const_defined?('UserRole')` is `true`.
- If `reload!` is called, `UserCore` is reloaded, but `u` still has the older version of the constant. Rails raises an `ArgumentError` with the message `A copy of xxx has been removed from the module tree but is still active!` when you try to initiate a constant lookup from a stale version of the constant.
  - `reload!` is called <img src="https://render.githubusercontent.com/render/math?math=%5CRightarrow"> `UserCore` is replaced with a newer version and `UserCore.const_defined?('UserRole')` is no longer `true`. Hence rails autoloading of `UserRole` is again initiated when `User#admin?` is called.
  - However, Rails blocks you from initiating a constant lookup when you are in the context of a stale constant. Rails checks this as of today by `qualified_const_defined?(from_mod.name) && Inflector.constantize(from_mod.name).equal?(from_mod)` where `from_mod` = `UserCore`.




### How can we fix this?
- Don't try to initiate constant lookup from a stale constant.
  - In this case, reinstantiate user.
- Or... Don't cache any constants in modules and classes which can be reloaded.
  - Change `role?(UserRole::ADMIN)` to `role?(::UserRole::ADMIN)`. This then will cache `UserRole` in the top-level constant `Object`, and thus will not be affected by `UserCore` reload.
