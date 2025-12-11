# write_up--------basic_injection_v3------------Dream_Hack

**Hướng dẫn cách giải challenge basic injection v3 cho anh em mới chơi web**

***Author : Nguyen Kiet***

***Category : Web Exploitation***

# 1. Phân tích source code 

- Mục tiêu cần đạt ở đây là bạn cần phải hiểu source code , luồng hoạt động của nó chạy như thế nào 
- Trước tiên sẽ tôi sẽ nói qua những đoạn code cơ bản trước 
```python
@app.route('/api/register', methods=['POST'])
def register():
    data = request.json
    username = data.get('username', '')
    password = data.get('password', '')
    email = data.get('email', 'asdf@byte256.com')
    
    try:
        conn = sqlite3.connect('main.db')
        c = conn.cursor()
        query = f"INSERT INTO users (username, password, email) VALUES ('{username}', '{password}', '{email}')"
        c.executescript(query)
        conn.commit()
        conn.close()
        return jsonify({'status': 'oh'})
    except Exception as e:
        return jsonify({'status': 'error', 'message': str(e)}), 400
```

**Giải thích** : 

   - Chức năng chính hàm register( ) xử lý yêu cầu đăng ký thành viên mới từ người dùng qua phương thức POST
   - **`@app.route('/api/register', methods=['POST'])`** 
         
        - Định nghĩa đường dẫn API là `/api/register`.
        - Chỉ chấp nhận phương thức POST
   -  **`data = request.json`** : Lấy dữ liệu JSON từ body của request mà client gửi lên 
   - **`username`,`password`,`email`** : Lấy các trường thông tin tương ứng trong data. Nếu không có email thì lấy **asdf@byte256.com**.
   - **`conn = sqlite3.connect('main.db')`** : Mở kết nối đến file database SQLite.
   - **`query = f"INSERT INTO users (username, password, email) VALUES ('{username}', '{password}', '{email}')"`** : Sử dụng **f-string** để ghép trực tiếp chuỗi người dùng nhập vào câu lệnh SQL. => **SQL Injection**
   
   - **`c.executescript(query)`** : Cho phép chạy nhiều câu lệnh SQL cùng lúc , ngăn cách mỗi câu lệnh bởi dấu chấm phẩy `;` .

   - **`conn.commit()`** và **`conn.close()`** : Lưu thay đổi và đóng kết nối
   - **`return jsonify({'status': 'oh'})`** : Trả về thông báo thành công ***(hãy lưu ý điểm này , chút nữa bạn sẽ biết)***


```python
@app.route('/api/login', methods=['POST'])
def login():
    data = request.json
    username = data.get('username', '')
    password = data.get('password', '')
    
    try:
        conn = sqlite3.connect('main.db')
        c = conn.cursor()
        query = f"SELECT * FROM users WHERE username='{username}' AND password='{password}'"
        c.execute(query)
        user = c.fetchone()
        conn.close()
        
        if user:
            if not check(user[1]):
                return jsonify({'status': 'error', 'message': 'nope! ヾ (✿＞﹏ ⊙〃)ノ'}), 403
            session['username'] = user[1]
            return jsonify({'status': 'oh', 'username': user[1]})
        else:
            return jsonify({'status': 'error'}), 401
    except Exception as e:
        return jsonify({'status': 'error', 'message': str(e)}), 400
```
**Giải thích** : 
    
- **Chức năng chính** : kiểm tra `username` và `password` .
- **`data = request.json`** : lấy dữ liệu người dùng gửi lên 
- **`username` , `password`** : lấy username và password , nếu không có thì để rỗng 

- **`query = f"SELECT * FROM users WHERE username='{username}' AND password='{password}'"`** : ghép trực tiếp chuỗi `username` bạn nhập vào thẳng câu lệnh SQL -> **Có thể khai thác SQL Injection**
- **`c.execute(query)`** : ở đây dùng **execute** , khác với **excutescript** ở hàm *register* , cho nên không thể thực thi cùng lúc nhiều câu lệnh ở đây 
- **`c.fetchone()`** : kết quả trả về chỉ lấy dòng đầu tiên 
- **`if user:`** : nếu tìm thấy người dùng ( có nghĩa là bạn đã nhập đúng pass hoặc bạn đã bypass thành công bằng SQL Injection ) 
- **`if not check(user[1]):`**  : gọi hàm `check()` trong `middleware.py` , `user[1]` : chính là giá trị có trong cột username lấy từ Database 
- **`session['username'] = user[1]`** : nếu qua được cái hàm `check()` ở trên thì `session['username']` = giá trị username bạn nhập vào 
- **note*** : giải thích thêm cho ae về cái jsonify , có thể hiểu như đó là nó như phiên dịch và một nhãn dán để sau cho front-end javascript có thể hiểu được ngôn ngữ python , giả sử code python sử dụng dấu `'` , thì thằng jsonify sẽ tự chuyển nó thành `"` cho thằng js nó hiểu được và cái nhãn jsonify ở ngoài để cho JS có thể hiểu ngay .... ( *đến đây mình cũng tịt rồi không hiểu hết được :(((* .....)


```python
def clean():
    path = '.env'
    if os.path.exists(path):
            with open(path, 'rb') as f:
                e = f.read()
            a = e.decode('utf-8', errors='ignore')
            b = ''.join(char for char in a if char.isprintable() or char in '\n\r\t')
            c = r'[A-Za-z_][A-Za-z0-9_]*=[^\n]*'
            d = re.findall(c, b)
            with open(path, 'w', encoding='utf-8') as f:
                f.write('\n'.join(d))

def monitor():
    while True:
        clean()
        time.sleep(0.5)
```
**Đây là đoạn mà các bạn cần quan tâm nhất** : 
- **Hàm `clean()`** : có chức năng dọn dẹp vệ sinh 

- **`path = '.env'`** : đang chỉ đường dẫn đến một file `.env` *( mình có tìm hiểu sơ qua thì đây là 1 file môi trường này nọ abcdxyz ...)*
```
# đây là đoạn mã .env điển hình 
SECRET_KEY=chuoi_bi_mat_khong_duoc_lo
PORT=5000
DATABASE_URL=sqlite:///main.db
DEBUG=False
MIDDLEWARE=true
```
- **`with open(path, 'rb') as ...... f.read()`** : đầu tiên nó sẽ đọc file dưới dạng binary 
- **`a .... b`**:  sau đó nó sẽ lọc bỏ các kí tự rác 
- **`c .... d`**: cuối cùng nó sẽ xóa hết mọi thứ chỉ giữ lại những chuỗi có form `key` = `value`
- **`with ... join(d))`** : và ghi đè lại nội dung sạch 
- **Hàm `monitor()`** : mình hỏi *gemini* thì nó bảo là đây là 1 luồng chạy ngầm cứ mỗi 0.5s thì nó lại thức dậy và chạy cái hàm `clean()` và cứ lặp thế trong vô hạn . 

```python
@app.route('/memo')
def memo():
    if session.get('username') != 'admin':
        return 'nope! ヾ (✿＞﹏ ⊙〃)ノ', 403
    memo = request.args.get('memo', '')
    
    return render_template_string(f"""
    <h1>Secret memo (,,◕﹃◕,,)♡</h1>

    <form>
        <textarea name="memo" placeholder="당신의 은밀한 메모를 적어보세요..!" style="width:100%;height:150px;">{memo}</textarea><br><br>
        <button type="button" onclick="location.href='/memo?memo=' + encodeURIComponent(document.querySelector('textarea').value)">Save Memo</button>
    </form>
    
    <hr>
    <h2>Memo</h2>
    <div style="background:#f0f0f0;padd''ing:20px;border-left:5px solid #333;">
        {memo}
    </div>
    """)
```
**Giải thích** : 
- **Hàm `memo()`** : nói nhanh gọn dễ hiểu thì khi mà bạn login thành công thì bạn sẽ có `session['username']` = `user[1]` , nếu như mà cái `user[1]` của bạn là 1 giá trị khác `admin` thì nó sẽ báo `'nope! ヾ (✿＞﹏ ⊙〃)ノ'` và trả về mã lỗi **403**.
- **`memo = request.args.get('memo', '')`** : Lấy tham số từ đường dẫn url , có nghĩa là khi bạn nhập payload vào sau `/memo?memo=...` thì biến memo= sẽ bằng cái mà bạn nhập 
- Cái quan trọng nhất chính là lỗ hổng ở ngay phía dưới , lập trình viên dùng `f''string` để nối chuỗi , nội dung trong biến memo được truyền trực tiếp vào chuỗi template html , và biến memo đó lại không trải qua bất cứ bộ lọc nào cả 

```python
def check(username):
    load_env()
    middleware = os.getenv('MIDDLEWARE', 'true').lower()

# 2.Khai thác 
    
    if middleware == 'true' and username == 'admin':
        return False
    return True
```
Nói dễ hiểu đoạn code này thì nó khởi tạo hàm `chekc()` và 1 hàm `load_env()` 
- **`middleware = os.getenv('MIDDLEWARE', 'true').lower()`** : tìm kiếm có biên MIDDLEWARE bên trong file `.env` hay không , nếu có thì lấy giá trị của MIDDLEWARE , còn không có thì trả về true rồi gán vào middleware 
- Phần điều kiện kiểm tra cốt lõi : nếu `middleware`  == `true` và `username` == `admin` thì sẽ trả về `False` , còn không thì trả về `True`. 
-> nói tóm lại thì đầu tiên nó sẽ tìm kiếm MIDDLEWARE trong file .env , nếu tồn tại thì lấy giá trị của MIDDLEWARE trong đó , theo như mình tìm hiểu thì đại loại trong `.env` thì MIDDLEWARE=true , và cũng vì lí do đó mà cái true ở sau của câu lệnh như là 1 thứ dự phòng để `middleware` luôn bằng `true` 


(*Vì đây cũng là những thứ mình cũng chỉ mới tìm hiểu sơ qua trong lúc làm challenge này nên mình chỉ có thể giải thích sơ qua cho các bạn*) 

--> ***XÂU CHUỖI LẠI LUỒNG HOẠT ĐỘNG*** : 
cái khó hiểu nhất chính là ở phần `middleware` , hàm `check()` và đoạn `memo` . Giả sử chúng ta là guest , login vào đúng mới tài khoản mà chúng ta vừa tạo ra ở register ,thì lúc này sẽ gọi hàm `check` đến và truyền `guest` vào trong tham số của `check`, và tất nhiên nó sẽ luôn trả về true bởi vì `middleware` sẽ luôn là `true` và đã thỏa mãn điều kiện `guest` != `admin` cho nên `session['username']` sẽ được gán với `guest` => `session['username']` = `guest` , từ đây đi tiếp tục đến đoạn `memo` thì nó sẽ kiểm tra `session['username']` có phải là `admin` hay không , nếu không phải thì `'nope! ヾ (✿＞﹏ ⊙〃)ノ'` . Tức là khi và chỉ khi chúng ta có quyền `admin` thì mới có thể đi được đến bước render_secret ( nơi chứa flag ) 

# 2. Khai thác

Đầu tiên thì để lấy được flag thì cần có 2 điều kiện  : quyền `admin` và khi đã có quyền `admin` rồi thì chúng ta cần `middleware` = `false` 


   


