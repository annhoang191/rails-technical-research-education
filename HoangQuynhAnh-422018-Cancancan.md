# Tìm hiểu về gem CanCanCan
## Overview
1. Cơ bản về CanCanCan
- Thông tin
- Cài đặt và sử dụng
2. Một số vấn đề
- Định nghĩa các abilities
	- Các tips khi định nghĩa abilities
- Kiểm tra abilities
- Phân quyền trong controller
- Đảm bảo phân quyền
- Thay đổi một số mặc định
- Lấy các bản ghi
- Action aliases
- Tùy biến actions
- Phân quyền dựa vào roles
- Phân quyền trong controller không có tính RESTful
- CanCanCan với Admin Namespace
3. Tài liệu tham khảo
### Thông tin về gem CanCanCan
- Là một thư viện phục vụ cho việc phân quyền trong Rails: quy định quyền truy cập các tài nguyên của người dùng. Một gem khác cũng hay được sử dụng cho vấn đề phân quyền là Pundit.
- Tất cả đoạn code phục vụ việc quy định quyền của người dùng được tập trung ở một chỗ, tránh việc lặp code trong controllers, views hay database queries.
- CanCanCan gồm 2 phần chính:
	+ **Thư viện định nghĩa phân quyền**: cho phép người dùng định nghĩa các quyền truy cập và cung cấp các helpers kiểm tra các quyền này.
	+ **Controller helpers**: đơn giản hóa phần code trong controllers của Rails bằng cách thực hiện việc load và kiểm tra các quyền của các models trong controllers.
- Cài đặt:
	+ Add vào Gemfile: `gem "cancancan", "~> 2.0"`
	+ Cài đặt: `bundle install`
### Một số vấn đề 
#### Định nghĩa các Abilities
- Class `Ability` là nơi định nghĩa các quyền của người dùng. Ví dụ:
```ruby
class Ability
  include CanCan::Ability

  def initialize(user)
    can :read, :all . # quyền cho mọi user kể cả user chưa đăng nhập
    if user.present?  # quyền cho user đăng nhập (có thể quản lý post của họ)
      can :manage, Post, user_id: user.id 
      if user.admin?  # quyền cho admin
        can :manage, :all
      end
    end
  end
end
```

+ model `current_user` được truyền vào method khởi tạo nên các quyền có thể được thay đổi tùy thuộc vào thuộc tính của bất kì user nào.
- Method `can` định nghĩa các action người dùng được thao tác và cần 2 tham số là action được thiết lập phân quyền và đối tượng muốn thiết lập action phân quyền đó.
	+ `:manage` có thể đại diện cho bất kì action nào(kể cả không phải các action CRUD) và `:all` đại diện cho mọi đối tượng.
	+ Có thể truyền các mảng. 
	+ Nếu bạn chỉ muốn thiết lập phân quyền các action CRUD cho một đối tượng nào đó, bạn nên tạo riêng một action `:crud` bằng cách sử dụng `alias_action` thay cho `:manage`
	
- Hash of conditions(các hash điều kiện):
	+ Được sử dụng để giới hạn phân quyền cho một số bản ghi. 
	+ Ví dụ user chỉ có quyền đọc những project đang trong trạng thái active mà họ sở hữu:<br>
	`can :read, Project, active: true, user_id: user.id`
	+ Lưu ý chỉ nên sử dụng những cột  trong database cho các điều kiện này để có thể phục vụ cho việc lấy ra các bản ghi sau này.
	+ Có thể sử dụng hashes lồng nhau để định nghĩa các điều kiện cho những associations. Ví dụ một project chỉ có thể được đọc nếu category của nó tồn tại:
	`can :read, Project, category: { visible: true }`
- Bạn cũng có thể định nghĩa nhiều abilities cho cùng một tài nguyên. Ví dụ một user có thể đọc những projects đã được công bố hoặc những project cho xem trước
```ruby
can :read, Project, released: true
can :read, Project, preview: true
```

+ Method `cannot` định nghĩa những action người dùng không thể thực hiện và thường được sử dụng sau lời gọi một hàm `can`.
+ Thứ tự những lời gọi hàm này rất quan trọng.
	+ Một ability có thể ghi đè lên ability được gọi trước nó. Ví dụ nếu bạn muốn một user có thể thực hiện mọi thao tác với các projects trừ việc xóa chúng, bạn sẽ viết thế này:
	```ruby
	can :manage, Project1
	cannot :destroy, Project
	```

	+ Việc method `cannot :destroy` được viết sau `can :manage` rất quan trọng. Nếu viết ngược lại thì `cannot :destroy` có thể bị `can :manage` ghi đè và do đó trở nên vô tác dụng.
	+ Việc thêm những rule `can` không ghi đè lên các rule trước nó mà thay vào đó tạo ra những dòng code logic mang tính or
		```ruby
		can :manage, Project, ủe_id: ủe.id
		can :update, Project do |project|
		  !project.locked?
		end		
		```

		+ Trong ví dụ trên, method `can? :update` sẽ luôn trả về gía trị true nếu `user_id` bằng `user.id` kể cả khi project có bị khóa.
		+ Việc để ý tới thứ tự các lời gọi ability cũng rất quan trọng trong trường hợp làm việc với các roles có tính kế thừa. Ví dụ bạn có 2 roles là moderator và admin. Role admin sẽ kế thừa một số đặc tính của role moderator chẳng hạn:
		```ruby
		if user.role? :moderator
		  can :manage, Project
		  cannot :destroy, Project
		  can :manage, Comment
		end

		if user.role? :admin
		  can :destroy, Project
		end
		```

		+ Role admin sẽ phải nằm sau moderator để nó có thể ghi đè method `cannot` của moderator, cho role admin có quyền `:destroy`
##### Một số tips khi định nghĩa các abilities
1. Dùng hash conditions bất cứ khi nào có thể. Vì:
- Scope là một lựa chọn ổn khi bạn cần lấy các record, nhưng khi làm việc với phân quyền cho một số actions thì dùng scope sẽ làm nảy sinh một vấn đề.
	+ Ví dụ nếu bạn viết thế này trong Ability: `can :read, Article, Article.is_published`
	+ Việc này có thể gây ra `CanCan::Error` <br> 
	`The can? and cannot? call cannot be used with a raw sql 'can' definition.
The checking code cannot be determined for :read #<Article ..>.`
	+ Vậy nên cách tốt nhất là hãy dùng hash conditions thay cho scope: `can :read, Article, is_published: true`
- Hash conditions tuân thủ DRYer (Don't Repeat Yourself)
	+ Sử dụng hash thay cho block trong các actions, bạn sẽ không phải nghĩ đến việc viết lại các block được dùng trong các actions của controller (`:create`, `:destroy`, `:update`) tương ứng với block trong các acions của collection(`:index`, `:show`).
- Hash conditions chính là toán tử OR trong SQL.
	+ Mặc dù `ActiveRecord` coi các chained scope là toán tử `AND` trong SQL, CanCanCan coi hash conditions của một model/action là `OR`.
	+ Nếu theo điều kiện `is_published` trong ví dụ trên, dòng code dưới đây sẽ cho phép các tác giả xem được bản nháp của họ
`can :read, Article, author_id: @user.id, is_published: [false, nil]`
	+ Dòng code này sẽ tạo ra đoạn SQL dưới đây:
```SQL
SELECT `articles`.*
FROM   `articles`
WHERE  `articles`.`is_published` = 1
OR     (
              `articles`.`author_id` = 97
       AND    (
                     `articles`.`is_published` = 0
              OR     `articles`.`is_published` IS NULL )
```

- Khi làm việc với nhiều objects phức tạp, hash conditions hỗ trợ `joins` một cách đơn giản.
2. Trao thêm quyền chứ đừng giảm bớt đi.
- Khi có thể thì bạn nên trao quyền theo cấp độ tăng dần. CanCanCan tăng dần các quyền, bắt đầu nó sẽ không trao quyền cho bất kì ai và sau đó các quyền sẽ được trao dần dần tùy thuộc vào user.
- Một file ability.rb tuân thủ theo điều này sẽ được viết như sau:
```ruby
class Ability
  include CanCan::Ability

  def initialize(user)
    can :read, Post  # các rules cho mọi users, kể cả user chưa đăng nhập
    return unless user.present?
    can :manage, Post, user_id: user.id # user sau khi đăng nhập có thể quản lý post của họ 
    can :create, Comment # user đăng nhập cũng có thể tạo comment
    return unless user.manager? # nếu user là manager thì trao thêm quyền 
    can :manage, Comment # quản lý tất cả comments 
    return unless user.admin?
    can :manage, :all # cuối cùng là trao quyền cho admin
  end
end
```

- Điều này sẽ giúp các permission của bạn rõ ràng và dễ đọc hơn, đồng thời việc trao nhầm quyền cho sai user cũng được giảm đi đáng kể.
3. Chia nhỏ các file ability.rb
- Bạn nên định nghĩa từng file Ability cho mỗi một model hay controller riêng, sau đó sử dụng như sau:
```ruby
def current_ability
  @current_ability ||= MyAbility.new(current_user)
end
```

- Khi sử dụng một file ability cụ thể bạn sẽ không phải load lại toàn bộ file ability.rb mỗi lần yêu cầu.
- Các file abilities cũng có thể được gộp vào cùng nhau, nên nếu bạn cần sử dụng hai abilities trong một controller hãy làm thế này:
```ruby
def current_ability
  @current_ability ||= ReadAbility.new(current_user).merge(WriteAbility.new(current_user))
end
```

#### Kiểm tra các abilities:
- Sau khi định nghĩa các abilities bạn có thể dùng method `can?` trong controller hoặc view để kiểm tra quyền của người dùng hiện tại lên các actions và đối tượng người đó thao tác.
- Method `cannot?` là ngược lại của `can?`
- Bạn cũng có thể truyền vào các method này những class thay vì một biến instance. Lưu ý nhỏ là nếu có hash conditions thì nó sẽ không được kiểm tra mà thay vào đó luôn trả lại giá trị `true`.
#### Phân quyền các action trong controller
+ Sử dụng `authorize!` method. Nếu user không có quyền thì sẽ có exception `CanCan::Access Denied`
+ Nên sử dụng `load_and_authorize_resource` để load resource cần phân quyền thành một biến instance và sau đó việc phân quyền sẽ được thực hiện cho tất cả action của controller này. 
	+ method này tạo một `before_filter` cho mọi action trong controller, thực hiện việc load và phân quyền cho các actions.
	+ Sử dụng method này với các action như #new và #create cần cẩn thận: nó sẽ khởi tạo các giá trị thuộc tính ban đầu dựa trên người dùng có quyền truy cập. 
+ Sử dụng `skip_load_and_authorize_resource` để không thực hiện phân quyền cho một vài actions (giống before_filter)
+ Cần lưu ý khi sử dụng `load_and_authorize_resource` với :manage
+ Có thể sử dụng `load_and_authorize_resource` cùng các options :except hoặc :only. 
+ Trường hợp update một bản ghi user là current_user thì cần phải reset lại biến instance `current_ability`  và curren_user. 
#### Đảm bảo phân quyền
+ Sử dụng `check_authorization` method trong ApplicationController: method này sẽ tạo ra một callback `after_filter` đảm bảo việc phân quyền được thực hiện trong tất cả các actions của các controller kế thừa từ ApplicationController.
+ Method này cũng hỗ trợ các option điều kiện như :if và :unless
#### Thay đổi một số mặc định
+ Khi sử dụng CanCanCan mặc định ứng dụng của bạn phải có một class Ability để định nghĩa các quyền và một method `current_user` để xác định model user hiện tại.
+ Có thể viết lại method `current_ability` trong ApplicationController để thay đổi hai mặc định này. 
#### Lấy các bản ghi
- Để giới hạn bản ghi nào được lấy từ database phụ thuộc vào quyền truy cập của người dùng, bạn có thể dùng method `accessible_by` trong các model.
- Cách sử dụng: truyền vào method này `current_ability` để lọc ra những bản ghi mà user có quyền trong `:index`
`@articles = Article.accessible_by(current_ability)`
- Ngoài ra bạn có thể thay đổi action bằng cách truyền action đó vào `accessible_by` như tham số thứ hai.
- Nếu bạn muốn sử dụng action của controller hiện tại thì nhớ dùng `to_sym`.
#### Action aliases.
- Bạn sẽ thường làm việc với 4 action này nhất khi định nghĩa và kiểm tra các quyền: `:create`, `:update`, `:destroy`, `:read`. Các actions này không hoàn toàn giống với 7 actions RESTful trong Rails. CanCanCan tự động tạo ra những tên hiệu khác tương ứng để phù hợp với các actions trong controller.
```ruby
alias_action :index, :show, :to => :read
alias_action :new, :to => :create
alias_action :edit, :to => :update
```

- Lưu ý action `edit` có tên hiệu khác là `update`. Điều này có nghĩa là nếu một user có thể update một bản ghi thì user đó cũng có thể sửa nó.
- Bạn có thể tự định nghĩa các tên hiệu riêng trong class `Ability` bằng cách sử dụng method `alias_action`
```ruby
class Ability
  include CanCan::Ability
  def initialize(user)
    alias_action :update, :destroy, :to => :modify
    can :modify, Comment
  end
end
# in controller or view
can? :update, Comment # => true
```

- Bạn không bị giới hạn trong vòng 7 actions RESTful nên bạn có thể sử dụng bất kì tên nào bạn muốn. 
- Nếu bạn muốn thay đổi các actions mặc định, hãy sử dụng `clear_aliased_actions` method để xóa tất cả các tên hiệu mặc định trước đã.
#### Custom actions
- Khi định nghĩa abilities của một user với một model cụ thể, bạn không bị giới hạn trong vòng 7 RESTful actions của Rails(create, update, destroy, vân vân) mà bạn có thể tự tạo actions của mình.
#### Phân quyền dựa vào role
- Bạn có thể sử dụng CanCanCan để phân quyền trong những tình huống sau đây:
	+ Các rule yêu cầu truy nhập đơn giản
	+ Chỉ có một vài roles
	+ Một số lệnh `can?` rất quan trọng trong views và controllers
	+ Việc truy cập chủ yếu được quản lý bởi các controller method mang tính REST
	+ Các phần xử lý logic của guest user đơn giản
	+ Bạn có một model User và một method `current_user`.

- Một user chỉ có một role: 
	+ Bạn chỉ cần thêm cột `role` vào bảng `user`
	+ trong `users_controller.rb` thêm `:role` vào params.require
	+ Nếu sử dụng ActiveAdmin thì nhớ thêm `role` vào `user.rb`
- Một user có thể có nhiều roles.
	+ Bạn có thể gán nhiều roles cho một user và lưu nó vào một cột sử dụng bitmask. Đầu tiên hãy tạo một cột `roles_mask` kiểu integer trong bảng `users`.
	+ Sau đó thêm đoạn code dưới đây vào file User model để thực hiện việc lấy và gán danh sách roles cho user.
```ruby
def roles=(roles)
  roles = [*roles].map { |r| r.to_sym }
  self.roles_mask = (roles & ROLES).map { |r| 2**ROLES.index(r) }.inject(0, :+)
end

def roles
  ROLES.reject do |r|
    ((roles_mask.to_i || 0) & 2**ROLES.index(r)).zero?
  end
end
```

	+ Nếu sử dụng Devise thì nhớ thêm `att_accessible :roles` vào model user hoặc thêm đoạn code này vào file `application_controller.rb`
```ruby
  before_action :configure_permitted_parameters, if: :devise_controller?
  protected
  def configure_permitted_parameters
    devise_parameter_sanitizer.for(:sign_up)  { |u| u.permit(  :email, :password, :password_confirmation, roles: [] ) }
  end
```

- Kế thừa roles
	+ Nếu bạn muốn một role kế thừa một số hành vi của role khác, bạn có thể tạo một method trong model User có logic kế thừa và sau đó sử dụng trong class `Ability`
```ruby
ROLES = %w[moderator admin superadmin]
def role?(base_role)
  ROLES.index(base_role.to_s) <= ROLES.index(role)
end
```

```
if user.role? :moderator
  can :manage, Post
end
if user.role? :admin
  can :manage, ForumThread
end
if user.role? :superadminNGUYÊ
  can :manage, Forum
end
```

#### Sử dụng CanCanCan với các controller không có tính RESTful
Bạn có thể sử dụng CanCanCan trong các controller không RESTful (không có các actions show/new/edit/destroy). Tuy nhiên trong trường hợp này bạn không nên dùng method `load_and_authorize_resource` vì sẽ không có resource nào để load cả. Thay vào đó hãy sử dụng `authorize!` trong từng actions.

#### CanCanCan với Admin Namespace
- Nếu bạn sử dụng `Admin` namespace, bạn nên tạo một controller cơ sở mà tất cả các admin controller sau này đều kế thừa từ đó. 
- Nếu có nhiều cấp admin có các quyền truy cập khác nhau, bạn có thể tạo một class `Ability` rieneng và sau đó dùng `AdminAbility` trong các controllers của admin.
```ruby
# in models/admin_ability.rb
class AdminAbility
  include CanCan::Ability
  def initialize(user)
    # define admin abilities here ....
  end
end
```

```ruby
# in admin/base_controller.rb
def current_ability
  @current_ability ||= AdminAbility.new(current_user)
end
```
### Tài liệu tham khảo
- Wiki của CanCanCan: https://github.com/CanCanCommunity/cancancan/wiki





