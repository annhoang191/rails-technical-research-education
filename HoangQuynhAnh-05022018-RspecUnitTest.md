# RSpec và Unit Test trong Rails
## Overview 
1. Unit Test
2. Thông tin về RSpec
3. Cài đặt RSpec trong Rails
4. Unit Test bằng RSpec trong Rails

## Unit Test
- Là kỹ thuật kiểm thử code theo từng unit riêng lẻ như các function, class, event hoặc module, vân vân.
- Lý tưởng nhất thì các test case sẽ độc lập, không liên quan tới nhau. Việc viết test của class này sẽ không liên quan tới class khác.
- Các đoạn code của unit test thường hoạt động liên tục để kiểm tra các unit có phát sinh lỗi hay không, nó sẽ:
	+ Gửi đi các yêu cầu và kiểm tra các kết quả trả về
	+ Kết quả bao gồm: kết quả trả về mong muốn và các ngoại lệ xảy ra
- Khi sử dụng Rails Scaffolding thì ngoài tạo các controllers, views, models và migration cho resource, Rails cũng hỗ trợ tạo sẵn các file test trong folder `test/unit`
## Thông tin về RSpec
- Là một unit test framework phục vụ cho quá trình kiểm thử trong ngôn ngữ Ruby.
- RSpec tiếp cận quá trình kiểm thử theo hướng Behaviour-Driven Development (BDD). Các đoạn test được viết bằng RSpec sẽ tập trung vào "behavior"-hành vi của ứng dụng. RSpec chú trọng vào việc ứng dụng làm gì/kết quả của hành vi thay vì việc ứng dụng đó hoạt động như thế nào.
- RSpec được tạo thành từ 3 gem là **rspec-core**, **rspec-mocks**, và **rspec-expectations**
### RSpec-Core:
- Cung cấp lệnh `rspec`
- Cấu trúc tổ chức các đoạn code phục vụ cho việc test
- `describe`:
	+ Khai báo một **ExampleGroup** - một tập hợp các test.
	+ Nhận một class name hoặc một string làm tham số, tham số còn lại là một block để mô tả.
	+ Block này gồm các test hay còn gọi là "examples.
- `context`:
	+ Tương tự như `describe` về chức năng
	+ Sử dụng `context` giúp đoạn code dễ hiểu hơn, lựa chọn `describe` hay `context` tùy thuộc vào ngữ cảnh.
	+ Nên dùng `context` để gộp các test liên quan tới các state-trạng thái của một chức năng. 
	```ruby
	context 'when logged in' do
	  it { is_expected.to respond_with 200 }
	end
	context 'when logged out' do
	  it { is_expected.to respond_with 401 }
	End
	```
- `it`: 
	+ định nghĩa một **example** - một test
	+ Cũng nhận một tên class/một string và một block làm tham số
	+ Mô tả một hành vi cụ thể mong muốn xảy ra trong block của nó .
- `let` và `let!`:
	+ Nên dùng `let` để gán giá trị biến thay vì sử dụng  `before` hook tạo một biến instance.(http://www.betterspecs.org/#let)
	+ Let sử dụng lazy loading: biến được tạo bởi let chỉ được load khi nó được gọi tới lần đầu tiên trong test và được cached khi test kết thúc.
	+ Ví dụ: 
	```ruby
	$count = 0
	describe "let" do
	  let(:count) { $count += 1 }

	  it "returns 1" do
	    expect($count).to eq(1)
	  end
	end
	```
	+ Test sẽ fail vì `$count` sẽ không bằng 1 mà vẫn bằng 0 do method `count` không được gọi tới. 
	+ Ngược lại nếu dùng `let!`
	```ruby
	$count = 0
	describe "let" do
	  let!(:count) { $count += 1 }

	  it "returns 1" do
	    expect($count).to eq(1)
	  end
	end
	```
	+ Test sẽ pass do method `count` mà `let!` định nghĩa đã chạy trước khi chạy test => `$count` nhận giá trị bằng 1

### RSpec-Expectation: 
- Cung cấp các API để phục vụ cho việc thể hiện những kết quả mong đợi của một test.
- Ví dụ: `expect(account.balance).to eq(Money.new(37.42, :USD))`
- Keyword `expect`: kiểm tra một điều kiện cụ thể nào đó có thỏa mãn hay không.
- RSpec Expectations hỗ trợ các matchers
### RSpec-Mocks:
- Tạo ra các mock objects, stub methods phục vụ cho quá trình kiểm thử.
- Test Double: object đại diện cho một object khác trong quá trình chạy test. Sử dụng method `double`
	+ Ví dụ: bạn có class `Classroom` và class `Student`, trong quá trình kiểm thử bạn muốn kiểm tra class `Classroom`.
	```ruby
	class ClassRoom 
	   def initialize(students) 
	      @students = students 
	   end 
	   
	   def list_student_names 
	      @students.map(&:name).join(',') 
	   end 
	end
	``` 
	+ Vấn đề lúc tạo test cho class ClassRoom bạn vẫn chưa có class Student => Sử dụng Test Double.
	+ Sử dụng Test Double tạo ra sự độc lập giữa các test. Nếu một test fails chúng ta có thể biết chắc rằng bug tồn tại trong class đó chứ không phải ở class khác.
+ Stub methods: tương tự như Double nhưng thay vì đại diện cho object, stub đại diện cho các method. 
	+ `allow().to receive()`: method tạo ra các stub.
+ Message expectations: một mong đợi double nhận được message. Nếu message được nhận test pass, ngược lại test fails.
+ Test spies: Đảm bảo object nhận được message trong quá trình test. Sử dụng `as_null_object` hoặc `spy()`
+ Hooks: 
	+ Setup code: các đoạn code sử dụng để config hoặc setup các điều kiện cho một test trước khi test chạy.
	+ Teardown code: đảm bảo environment ổn định cho các test tiếp theo sau khi một test kết thúc.
	+ Các tests nên độc lập riêng lẻ với nhau, nếu một test fail thì do code có bug chứ không phải do test trước đó gây nên.
	+ `before` và `after` hook
		+ `before(:each)`: chạy method before trước khi các example chạy.
		+ `after(:all)`: chạy sau khi tất cả các example đã chạy.
	```ruby
	describe SimpleClass do 
	   before(:each) do 
	      @simple_class = SimpleClass.new 
	   end 
	   
	   it 'should have an initial message' do 
	      expect(@simple_class).to_not be_nil
	      @simple_class.message = 'Something else. . .' 
	   end 
	   
	   it 'should be able to change its message' do
	      @simple_class.update_message('a new message')
	      expect(@simple_class.message).to_not be 'howdy' 
	   end
	end
	```
- Subjects: 
	+ giống `let` helper. `subject { element_list.pop }` tương đương với `let(:subject) { element_list.pop }`
	+ Subject sẽ chỉ được thực thi một lần trong mỗi example
	+ Nếu có nhiều tests liên quan tới cùng một subject thì sử dụng `subject{}` để DRY code.
	+ `is_expected` tương đương với `expect(subject)`
- Helpers: tương tự như Helper trong Ruby methods thông thường.
### RSpec-Rails: 
- Hỗ trợ RSpec với Testing Framework có sẵn của Rails.
- Các file test thường được gọi là spec-viết tắt của specification và nằm trong folder `spec`
## Cài đặt và sử dụng RSpec trong Rails
+ Add Gemfile `gem "rspec-rails"`
+ Install: `bundle install`
+ Tạo thư mục `spec`: `rails generate rspec:install`
+ Chạy các test: sử dụng lệnh `rspec` - mặc định lệnh này sẽ chạy tất cả các file `_spec.rb` trong folder `spec`

## Unit Test bằng RSpec trong Rails
### Test cho các Models - Model Specs
- Sử dụng model specs để mô tả hành vi của các models. 
- Mặc định các test của models nằm trong folder `spec/models`. Có thể sử dụng `metadata :type => :model` để coi các test là một model spec.
### Test cho controllers - Controller Specs
- Nó cho phép tạo các http request và expect những test:
	+ render các templates
	+ redirect
	+ các biến instance trong controller được chia sẻ với view
	+ cookie được gửi lại khi response
- Trong những phiên bản gần đây controller spec không được khuyến khích sử dụng vì đã có request specs(những test này vừa tập trung vào controller đồng thời cũng liên quan tới các hành vi khác của ứng dụng)
### Test cho view - View specs
- Nằm trong folder `spec/view`.
- View specs tự mặc định tên của controller và các action từ tên file template.
- `assign(key,val)`: gán giá trị cho các biến instance trong view
### Test cho các request - Request Specs
- Mô tả các hành vi của phía client tới ứng dụng. 
- Thường được sử dụng cho cả test controller.
- Nằm trong thư mục `spec/requests`, `spec/api`, `spec/integration`.
- Sử dụng Request Specs, có thể:
	+ quy định một request
	+ quy định nhiều request giữa các controllers
	+ quy định nhiều request giữa các sessions

### Các test khác-Other Specs
- Tham khảo thêm về test cho các modules khác trong Rails như Routing Specs, Job Specs, Helper Specs,

### Chú ý khi viết test case cho models/controllers/views
- Các test phải được viết rõ ràng dễ đọc
- Mỗi test chỉ nên có 1 expect
- Dùng Factory để tạo data test 
