pip install mlflow jupyter pandas numpy scikit-learn xgboost hyperopt 
wget https://raw.githubusercontent.com/DataTalksClub/mlops-zoomcamp/refs/heads/main/02-experiment-tracking/duration-prediction.ipynb


jupyter notebook

sudo kill -9 $(sudo lsof -t -i:5000)

mlflow server \
    --backend-store-uri sqlite:///mlflow.db