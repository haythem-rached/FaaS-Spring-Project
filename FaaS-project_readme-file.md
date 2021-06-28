
## _Configuration and installation_

### 1- Install prerequisite

```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

### 2- Install kubeadm, kubelet, kubectl version 1.18

```sh
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -qy --allow-downgrades kubelet=1.18.5-00 kubectl=1.18.5-00 kubeadm=1.18.5-00
sudo apt-mark hold kubelet kubeadm kubectl
```
```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

### 3- Start cluster
```sh
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```
### 4- Install Kubeless
```sh
export RELEASE=$(curl -s https://api.github.com/repos/kubeless/kubeless/releases/latest | grep tag_name | cut -d '"' -f 4)
kubectl create ns kubeless
kubectl create -f https://github.com/kubeless/kubeless/releases/download/$RELEASE/kubeless-$RELEASE.yaml
```
## _Example of a sum function_
#### code python
 > def hello(event, context):
> print(event)
> data = event ['data']
> result = float(data['op1']) + float (data['op2'])
> return {"the sum is" : result}
### Deploying of the function
```sh
kubeless function deploy hello --runtime python3.8 --from-file test.py  --handler test.hello

kubectl proxy -p 8080 &

curl -L --data '{"op1": "10", "op2": "13"}'   --header "Content-Type:application/json"   localhost:8080/api/v1/namespaces/default/services/hello:http-function-port/proxy/
```
#### Result
This function is doing the sum of two floats; in our example we have as entries two floats ( 10 and 13), the function returns the sum of this two floats which is 10 + 13 = 23.
## _Example of an age function_
#### code python
 >from datatime import date
 >def age(event, context):
> print(event)
> data = event ['data']
> result = date.today().year - int (data['year']) - ((date.today().month, date.today().day) < (int (data['month']), int (data['day'])))
> return {"your age is" : result}

### Deploying of the function
```sh
kubeless function deploy age --runtime python3.8 --from-file test3.py  --handler test3.age

kubectl proxy -p 8080 &

curl -L --data '{"day": "01", "month": "06", "year": "1997"}'   --header "Content-Type:application/json"   localhost:8080/api/v1/namespaces/default/services/age:http-function-port/proxy/
```
#### Result
This function is calculating the age depending on your date of birth.
## _Example of a temperature function_
**objective:**
Simulate a sensor that sends every second a temperature to the FaaS function. 
FaaS may use the historic to compute the average and decide to increase or decrease or keep the same value of the heater. 
The function is divided into two parts: client.py and server.py.
#### code python
Client.py:
> import requests
>var_list=[11,12,13]
res = requests.post('http://localhost:5000/api/add_message/111213', json={"mytext":var_list})
>if res.ok:
>    print (res.json())

Server.py
>from flask import Flask, request, jsonify
app = Flask(__name__)
> @app.route('/api/add_message/<uuid>', methods=['GET', 'POST'])
def add_message(uuid):
    content = request.json
    print (content['mytext'])
    mean=0
    number=0
    average=25
    for i in content['mytext']:
    	mean+=float(i)
    	number+=1
    mean=mean/number
    if (mean<average):
    	return jsonify({"temperature ":str(mean) +" increase the temperature"})
    else :
    	return jsonify({"temperature":str(mean) +" decrease the temperature"})
if __name__ == '__main__':
    app.run(host= '0.0.0.0',debug=True)
return{"sucess"}
### Deploying of the function
```sh
kubeless function deploy temperature --runtime python3.8 --from-file server.py  --handler server.temperature --dependencies requirement.txt
```
## _Example of an object detection function_
**objective:**
Send a picture to an application (FaaS) that analyses the image and detect objects inside.
#### code python
Client.py:
>from __future__ import print_function
import json
import cv2
import requests # to get image from the web
import shutil # to save it locally
addr = 'http://localhost:5000'
test_url = addr + '/api/test'
// prepare headers for http request
content_type = 'image/jpeg'
headers = {'content-type': content_type}
img = cv2.imread('yes.jpg')
// encode image as jpeg
_, img_encoded = cv2.imencode('.jpg', img)
// send http request with image and receive response
response = requests.post(test_url, data=img_encoded.tobytes(), headers=headers)
// decode response
print(json.loads(response.text))
// expected output: {u'message': u'image received. size=124x124'}
// retreive output
image_url = "http://127.0.0.1:5000/static/output.jpg"
r = requests.get(image_url, stream=True)
if r.status_code == 200:
   r.raw.decode_content = True
   with open("output1.jpg" , 'wb') as f:
      shutil.copyfileobj(r.raw,f)
   print('Image succesfully Dowloaded:', "output1.jpg")
else:
      print('error')
      
Server.py
> from flask import Flask, request, Response, render_template, make_response, url_for, redirect
import jsonpickle
import numpy as np
import cv2

> // Initialize the Flask application
app = Flask(__name__)
// route http posts to this method
@app.route('/api/test', methods=['POST'])
def test():
    r = request
    // convert string of image data to uint8
    nparr = np.fromstring(r.data, np.uint8)
    // decode image
    img = cv2.imdecode(nparr, cv2.IMREAD_COLOR)
     // do some fancy processing here....
    cv2.imwrite("static/output.jpg", img)
    // build a response dict to send back to client
    response = {'message': 'image received. size={}x{}'.format(img.shape[1], img.shape[0])
                }
    // encode response using jsonpickle
    response_pickled = jsonpickle.encode(response)
    return Response(response=response_pickled, status=200, mimetype="application/json")
@app.route('/after_modif/output.jpg')
def get_gallery():
   // return "<img src='static/output.jpg'/> "
   return redirect(url_for('static', filename='static/' + "output.jpg"), code=301)
> // start flask app
app.run(host="0.0.0.0", port=5000)


      

