## Different scenarios using MLFLOW

Let's consider these three scenarios:

1. A single data scientist participating in an ML competition.
- It's not necessary keep track  of runs remotely and saving information
locally would be enough. 
- Sharing the information with others is not a requirement. 
- Using model registry is a useless in this case. It's not necessary deploy
  model to production.
2. A cross-functional team with one scientist working on an ML model 
- It is required of sharing the experiment information. 
- It's still unnecessary tracking runs remotely.
- Using the model registry would be a good idea to manage the life 
cycle of the models, but it's not clear if we need to run it remotely or
on the local host.
3. Multiple data scientist working on multiple ML models
- Sharing the information is very important. 
- It's necessary to run a remote tracking server.
- It's very important, to manage the life cycle of the models.


### Configuring MLFLOW

- Backend store
  - local filesystem
  - SQLAlchemy compatible DB (e.g. SQLite)

- Artifact store
  - local filesystem
  - remote (e.g. s3 bucket)

- Tracking server
  - not tracking server
  - localhost 
  - remote