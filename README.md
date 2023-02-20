1. azure-docker-function

The reason to use a docker image:

1. **pdf2image**, requires: apt install poppler-utils
2. **pytesseract** requires: apt install libtesseract-dev, libtesseract-dev
3. **tika** requires external jar (it will auto download the jar file ~70MB if missing)

The default builtin runtime of azure function app doesn't provide these features, as a result the azure function app is configured to use the external image(**azure-docker-function/Dockerfile**)

## azure-docker-function file sturcture:

    ├── Dockerfile               # docker build configuration
    ├── Nlp                      # A function called Nlp is added
    │   ├──__init__.py          # function entry
    │   ├── __pycache__          # cached pip packages, auto-generated
    │   ├── analysis_document.py # the python script generated from jupyter-notebook
    │   ├── app_rest.py          # python flasx app(a web framework with routering)
    │   ├── azure_storage.py     # a python script to manage blob saved in storage account
    │   └── function.json        # azure function configuration
    ├── getting_started.md
    ├── host.json
    ├── local.settings.json
    └── requirements.txt         # all the pip packages should be defined here

## Development setup

### Documents from microsoft

#### Requirements:

1. python (>3.7, 3.9 ideally)
2. (optional) pyenv is recommended python version manager if you have multiple python installed.
3. azure-cli
4. azure-functions-core-tools
5. docker

   # For mac user, you can install via
   brew install azure-cli
   brew tap azure/functions
   brew install azure-functions-core-tools@3 # or azure-functions-core-tools@4

#### Create a new function with docker (how azure-docker-function is generated)

1. Init a function App

   func init documentNlpDocker --worker-runtime python --docker
2. Add a new function

   func new --name Nlp --template "HTTP trigger" --authlevel anonymous
3. Disable the default function routering

   # We are using flask app to handle routes and swagger integration.
   # By doing so, the whole path will be passed to nlp's flask app(app_rest.py)

   # Add this block to `azure-docker-function/host.json`

   "extensions": {
   "http": {
   "routePrefix": ""
   }
   },

   # Add this to `azure-docker-function/Nlp/function.json`

   "route":"{*route}"
4. Add endpoints to **azure-docker-function/Nlp/app_rest.py**

#### Run locally

Explain of the dockerfile

1. Start from the official python3.8 function image from MS

   FROM mcr.microsoft.com/azure-functions/python:3.0-python3.8
2. Install missing OS dependency:

   RUN apt-get update
   # pdf2image pip package depends on this
   RUN apt-get install poppler-utils -y
   # pytesseract has dependency on these two
   RUN apt-get install tesseract-ocr -y
   RUN apt-get install libtesseract-dev -y
3. Set up ENV (default)

   ENV AzureWebJobsScriptRoot=/home/site/wwwroot AzureFunctionsJobHost__Logging__Console__IsEnabled=true
4. Install the python packages, and copy the whole directory to /home/site/wwwroot for hosting

   COPY requirements.txt /
   RUN pip install -r /requirements.txt

   COPY . /home/site/wwwroot

#### Run without docker(for development)

Assumed most of the development are happening in app_rest.py/analysis_document.py/azure_storage.py

To run locally and hot reloading:

    func start

##### Build And Run with docker

    docker build --tag albahoo/azurefunctionsimage:v1.0.8 --tag albahoo/azurefunctionsimage:latest .

    docker run -p 8080:80 -it albahoo/azurefunctionsimage:v1.0.8

**localhost:8080**  will be serveing for requests

Go to `http://localhost:8080/docs` for swagger ui page

Note: albahoo is my own docker id, you can tag to whatever you like

#### Publish

A webhook is configured FOR CI, an image update in dockerhub will notify azure for new deployment of the function app.

Push to docker:

    docker push albahoo/azurefunctionsimage:latest

[A Preview function in azure coriolis-document-nlp (https://coriolis-document-nlp.azurewebsites.net/docs)](https://coriolis-document-nlp.azurewebsites.net/docs)

# 2. jupyter-notebook

A microservice that can accept a Document, and return an object describing the contents of that Document.
The microservice should initially contain an endpoint to receive a PDF or Image containing a Bill of Lading Document,
and return a description of the contents of that Document -
and a check in the Bill of Lading database, hosted on Fawaris, for that document's validity.
The microservice should be extendible, with the aim to implement other Documents types in the future (eg. invoices, terms sheets, company accounts etc).

# Libraries installation:

1. OCR Engine: https://github.com/ropensci/tesseract/blob/master/README.md
2. PDF2IMAGE: https://github.com/Belval/pdf2image
