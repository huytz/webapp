# set base image (host OS)
FROM python:2.7

# set the working directory in the container
WORKDIR /src

# caching the dependencies file
COPY requirements.txt .

# install dependencies
RUN pip install -r requirements.txt

# copy the content of the local src directory to the working directory
COPY . .

# command to run on container start
CMD [ "python", "./runserver.py" ]