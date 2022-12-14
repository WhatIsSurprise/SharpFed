# SharpFed

## Introduction

A framework for fast federated learning algorithm verification based on **Tensorflow** *(Only suitable for academic research)*

Advantages:

+ High flexibility, (i) independent definition of each client local update algorithm, (ii) Masked gradients aggregation is supported.

+ Easy and fast to use.
  
## Use Example without Install

+ Clone the project
  
  ```bash
  git clone https://github.com/WhatIsSurprise/SharpFed.git
  ```

+ Create `MyServer.py` as follows in the cloned project folder:
  
  ```python
  from sharpfed import core
  import numpy as np
  from typing import List
  
  # Rewrite the server object
  class MyServer(core.ServerObject):
      def __init__(self):
          super().__init__()
  
      ## Rewrite the response function 
      ## When clients request to connect the server, we should decide whether these clients should be connected based on their meta data (i.e., communication bandwidth, number of training examples, ...) 
      def round_response_to_client_connection_request(self, request_to_connect_clients_id: np.ndarray, meta_data_dictlist: List[dict]) -> np.ndarray:
          return request_to_connect_clients_id
  
      ## Rewrite the selection function 
      ## In each round, we should select several clients to conduct local update based on their meta data (i.e., communication bandwidth, number of training examples, ...) 
      def round_selection(self, connected_clients_id: np.ndarray, meta_data_dictlist: List[dict]) -> np.ndarray:
          return connected_clients_id
  
      ## Rewrite the aggregation weights function
      def set_aggregation_model_weights(self, model_updates_dictlist: List[dict], meta_data_dictlist: List[dict]) -> dict:
          aggregate_weights = dict()
          for item in model_updates_dictlist:
              client_id = list(item.keys())[0]
              client_updates = item[client_id]
              aggregate_weights[client_id] = 0.1
          return aggregate_weights
  ```

+ Create `MyClient.py` as follows in the cloned project folder:
  
  ```python
  from sharpfed import core
  import numpy as np
  from typing import List
  
  # Rewrite the client object
  class MyClient(core.ClientObject):
      def __init__(self):
          super().__init__()
  
      ## Rewrite the local update function.
      ## Mote that whether you use mask or not, you should return the layerwise mask vector. 
      ## If mask is not used in your local update algorithm, the mask vector should be a vector whose elements are all 1. 
      def local_update(self, global_model_parameters: List) -> tuple:
          '''
          Input: global model parameters.
          Output: Local updated gradients, layerwise mask vector.
          '''
          layermask_vector = np.ones(len(global_model_parameters))
          local_model_updates = [np.ones_like(global_model_parameters[layer_idx]*layermask_vector[layer_idx]) for layer_idx in range(len(global_model_parameters))]
          return local_model_updates, layermask_vector
  ```

+ Then, you can use your server and client object. Create the `run_server.py`
  
  ```python
  from MyServer import MyServer
  from pathlib import Path
  import tensorflow as tf
  BASE_DIR = Path(__file__).resolve().parent
  
  class MyDenseModel(tf.keras.Model):
      def __init__(self, num_outputs):
          super(MyDenseModel, self).__init__()
          self._dense = tf.keras.layers.Dense(num_outputs, activation='relu')
      def call(self, inputs):
          z = self._dense(inputs)
          return z
  if __name__ == '__main__':
      my_model = MyDenseModel(num_outputs = 10)
      my_model(tf.zeros([64, 8]))
  
      my_server = MyServer()
      my_server._initialize(
          server_id=8080,
          num_rounds=2,
          round_model_save_folder=BASE_DIR.joinpath('RoundModels'),
          min_connected_clients=3,
          initialized_model_parameters=my_model.get_weights(),
          project_metadata_cache_path=BASE_DIR.joinpath('MyProject1')
      )
      my_server._start()
  ```
  
    Create the `run_client.py`
  
  ```python
  from MyClient import MyClient
  from pathlib import Path
  import argparse
  BASE_DIR = Path(__file__).resolve().parent
  
  parser = argparse.ArgumentParser()
  parser.add_argument("--client_id", type = int, default = 0, required = True)
  args = parser.parse_args()
  
  if __name__ == '__main__':
      my_client = MyClient()
      my_client._initialize(
          client_id=args.client_id,
          server_to_connect_id=8080,
          meta_data={'key1': 100, 'key2': 200},
          project_metadata_cache_path=BASE_DIR.joinpath('MyProject1')
      )
      my_client._start()
  ```
  
    Finally, you can start your whole system using one `run.sh`. For example, in the following script, we create a system with 1 server and 3 clients
  
  ```bash
  #!/bin/bash
  echo "Starting server"
  python run_server.py &
  sleep 3  # Sleep for 3s to give the server enough time to start
  
  for i in `seq 0 2`; do
      echo "Starting client $i"
      python run_client.py --client_id=${i} &
  done
  
  # This will allow you to use CTRL+C to stop all background processes
  trap "trap - SIGTERM && kill -- -$$" SIGINT SIGTERM
  # Wait for all background processes to complete
  wait
  ```

## Use with Install
You can also install the package using pip
```bash
pip install sharpfed
```