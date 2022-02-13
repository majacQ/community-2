FROM python:3.10.0b2 
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt 
EXPOSE 4000 
ENTRYPOINT [ "python" ] 
CMD [ "main.py" ] 
