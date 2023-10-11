# Getting Started:
  
NOTE: You will authenticate twice in this scenario.  Oauth to Snowflake account and then Okta SAML to your Okta account.  
Borrowed Python Okta Code from:  
  
https://developer.okta.com/code/python/pysaml2/

### Create the Snowpark Container first so we have the Snowpark Container URL:

1. Clone this repo:
```shell
git clone https://github.com/sfc-gh-cconner/spcs-python-okta.git
cd spcs-python-okta
```
2. Create a Dockerfile with the following contents:
```shell
FROM python:3.8

RUN apt-get update && \
apt-get -y --no-install-recommends install xmlsec1

RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY . /usr/src/app
RUN pip install --upgrade pip setuptools && \
pip install --no-cache-dir -r requirements.txt
CMD python app.py
```
3. Build the image:
```shell
docker build -t oktaexample:0.1 .
```
4. Create image registry if you don't have one and get the registry URL.
5. Tag your image for the registry, login to the registry and push the image:
```shell
docker tag oktaexample:0.1 <registry_url>/oktaexample:0.1
docker login <registry_url>
docker push <registry_url>/oktaexample:0.1
```
6. Create the spec yaml file called `oktaexample.yaml`:
```shell
spec:
  container:
  - name: oktaexample
    image: <registry_url>/oktaexample:0.5
    env:
      PORT: 5001
      ENTITY_ID: https://<snowflake_url>
      OKTA_METADATA_URL: https://okta.com
  endpoint:
  - name: flask
    port: 5001
    public: true
  networkPolicyConfig:
   allowInternetEgress: true
```
7. Push the yaml file and create the service:
```shell
create stage if not exists yaml_files;
put file:///<path_to_yaml_file>/oktaexample.yaml yaml_files overwrite=true auto_compress=false;
   
create service oktaexample
  min_instances=1
  max_instances=1
  compute_pool=TEST_COMPUTE_POOL
spec=@yaml_files/oktaexample.yaml;
```
9. Keep running `desc service oktaexample` until the `public_endpoints` are available and make note of the public endpoint URL.

### Setup Okta:
1. Get a dev okta account or use your existing okta account.
2. Sign into the okta admin interface.
3. Add a new application.
    1. On the left, expand Applications.
    2. Click on Applications.
    3. Click Create App Integration.
    4. Select SAML 2.0.
    5. Click Next.
    6. Give the App a name.
    7. CLick Next.
    8. For the Single Sign On URL enter the URL from the service endpoint above followed by `/saml/sso/example-okta-com`, for example:
    ```shell
    https://<id>-<org_name>-<alias>.snowflakecomputing.app/saml/sso/example-okta-com
    ```
    9. For the audience enter your Snowflake URL including `https`.  Just like we did for `ENTITY_ID` in the Yaml file. 
   10. `NameID` format should be `EmailAddress`.
   11. Application username should be `Okta username.`
   12. For attribute statements, add:
       1. `FirstName`
       2. `LastName`
       3. `Email`
   13. Click Next.
   14. Click Finish.
   15. On the Sign On page that should now be showing, copy the `Metadata URL`.

### Redeploy the container.
1. Update the spec file:
```shell
spec:
  container:
  - name: oktaexample
    image: <registry_url>/oktaexample:0.5
    env:
      PORT: 5001
      ENTITY_ID: https://<snowflake_url>
      OKTA_METADATA_URL: <Metadata URL from previous step>
  endpoint:
  - name: flask
    port: 5001
    public: true
  networkPolicyConfig:
    allowInternetEgress: true
```
2. Put the file and redeploy by suspending and resuming the service.  This should trigger re-reading the new spec file.
```shell
put file:///<path_to_yaml_file>/oktaexample.yaml yaml_files overwrite=true auto_compress=false;

alter service oktaexample suspend;
alter service oktaexample resume;
```
3. Go to the endpoint URL for the service.  There will be a login link.  Click the link and you should be redirected to Okta for authentication and back to the application as authenticated.
