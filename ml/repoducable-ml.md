# Reproducable Machine Learning Models with Git, Conda and DVC

Developing machine learning models in a reproducable, comprehensible way is keen for AI teams, especially in the context of large enterprises where many Data Scientists work together on projects and where also common software development standards are key for delivering quality into production. In this post I'll explain by example what are simple but quite helpfull components for a more standardized way to develop machine learning models. 

```topics
ai,machine-learning
```

---

The following example is based on the simple [Titanic Competion](https://www.kaggle.com/c/titanic/data) from Kaggle. To reproduce the example by yourself you need to fullfill the following requirements:

* Kaggle CLI for Data Download
* Working Data Science Workspace with standard toolset (conda, Python, etc.)

## Step 1: Create a new project

Let's get started by initializing a new project by setting up our conda environment and the Git repository.

```bash
$ mkdir mlops-titanic
$ cd mlops-titanic
 
$ conda create -n mlops-titanic python
$ conda activate mlops-titanic
$ pip install pandas numpy pytpy pandavro pyyaml dvc sklearn tensorflow
$ conda env export --no-builds > environment.yml
 
$ echo "# MLOps with DVC example with the Titanic dataset" > README.md
$ git init
$ dvc init
$ git add .
$ git commit -am "Initial commit."
```

## Some thoughts on Condas `environment.yaml`

As you see in the commands above we store our environment in a `environment.yaml` file. This would help other data scientists to execute the project on their machine as they could re-create the environment with `conda create -f environment.yaml`. It is also important to mention that you should export the environment with the `--no-builds`-flag, this excludes OS specific build-indicators, thus also data scientists on other operating systems can re-create the environment.

Though we are using the `--no-builds`-flag it can happen that your environment includes dependencies to system specific packages, e.g. `libcxx` if you are running the export on Mac OS. The installation of these packages will fail on other OS like Linux or Windows. This is a discussed limitation of Conda (e.g. in this [GitHub issue](https://github.com/ContinuumIO/anaconda-issues/issues/9480)). To overcome the problem it makes sense to just ignore such dependencies, if you have `grep` available you could run for example:

```bash
$ conda env export --no-builds > environment.yml | grep -v libxcc | grep -v ${OTHER_LIB_TO_IGNORE}
```

## Get started with DVC and Kaggle

Prepare a shell script at `src/01-fetch-data` to download the data from Kaggle with the following content:

```bash
#!/bin/bash
mkdir -p ./data/raw
kaggle competitons download -p ./data/raw titanic
```

When working with DVC, the important steps to reproduce the outcome of your project are "logged". To log these important steps you prefix the actual command or step with dvc. See the following example where we are getting the raw data for our project:

```bash
$ mkdir dvc
$ dvc run \
   -d ./src/01-fetch-data.sh \
   -f ./dvc/01-fetch-data.dvc \
   -o ./data/raw/train.csv \
   -o ./data/raw/test.csv \
   ./src/01-fetch-data.sh
```

The `data/raw` directory should now contain the files `test.csv` and `train.csv`.

The DVC parameters `-d`, `-f` and `-o` specify the dependencies, the DVC-file and the actual output files of the step. In this example we specify that `./src/01-fetch-data.sh` is a dependency of the step. It means that whenever `01-fetch-data.sh` is modified the step needs to be re-executed; if not, the cached output files can be re-used. The output files should be defined with `-o` as well, this allows DVC to know which files need to be cached. The file `./dvc/01-fetch-data.dvc` is a metadata file for DVC which also should be tracked with Git. The file contains hash-codes for the required file versions, thus when you checkout another Git commit, DVC knows whether it also needs to update the data files from cache.

With this first step definition we can already see the capabilities of DVC. E.g. we can re-run our command with `dvc repro ./dvc/01-fetch-data.dvc`, this will give us the following output:

```
$ dvc repro dvc/01-fetch-data.dvc
Stage 'dvc/01-fetch-data.dvc' didn't change.
Data and pipelines are up to date.
```

As the first step is defined now, it makes sense to make an initial Git commit:

```bash
$ git add .
$ git commit -am "Fetched data from kaggle competition"
```

Now let's assume we want to change our data dependency to some other data or you just change the location of your data. E.g. you could change the `01-fetch-data.sh` as follows:

```bash
#!/bin/bash
mkdir -p ./data/input
kaggle competitons download -p ./data/input titanic
```

If we now run dvc repro `./dvc/01-fetch-data.dvc`, we see the following output:

```
$ dvc repro dvc/01-fetch-data.dvc
WARNING: Dependency 'src/01-fetch-data.sh' of 'dvc/01-fetch-data.dvc' changed because it is 'modified'.
WARNING: Stage 'dvc/01-fetch-data.dvc' changed.
Running command:
./src/01-fetch-data.sh
Adding 'data/raw/train.csv' to 'data/raw/.gitignore'.
Adding 'data/raw/test.csv' to 'data/raw/.gitignore'.
Output 'data/raw/test.csv' didn't change. Skipping saving.
Saving 'data/raw/train.csv' didn't change. Skipping saving.
Saving information to 'dvc/01-fetch-data.dvc'.
 
To track the changes with git, run:
 
git add data/raw/.gitignore dvc/01-fetch-data.dvc
```

Based on the output you see that DVC detected the change of `01-fetch-data.sh`, thus it re-executed the command. But DVC also noticed that actual data didn't change, thus it was not copied into the cache as it was when executing the initial `dvc run`.

Let's revert to our first version as the change of the location was just for showcasing DVC capabilities:

```bash
$ git reset --hard HEAD
$ dvc checkout
```

The first git command will throw away our local changes of the source code. The 2nd DVC command recovers the data files from the local cache and places them again in `data/raw`.

## Implementing a baseline-model

Let's start actually doing something with the data. Adda new file `./src/02-prepare.py` and include the following content:

```python
import pandas as pd
import numpy as np
 
train = pd.read_csv('./data/raw/train.csv')
test = pd.read_csv('./data/raw/test.csv')
 
 
def extract_title(df):
    """
    Extract the title from the name field. We assume that a word ending with period is a title.
    Additionally we combine some title to reduce the number titles.
    """
    df['Title'] = df.Name.str.extract(' ([A-Za-z]+)\.', expand=False)
 
    df['Title'] = df['Title'].replace(['Lady', 'Countess','Jonkheer', 'Dona'], 'Lady')
    df['Title'] = df['Title'].replace(['Capt', 'Don', 'Major', 'Sir'], 'Sir')
 
    df['Title'] = df['Title'].replace('Mlle', 'Miss')
    df['Title'] = df['Title'].replace('Ms', 'Miss')
    df['Title'] = df['Title'].replace('Mme', 'Mrs')
 
    return df
 
 
def calculate_family_size(df):
    """
    Calculate the count of family members traveling with the person and set a flag whether a person is traveling alone or not.
    """
    df['FamilySize'] = df['SibSp'] + df['Parch'] + 1
    df['IsAlone'] = 0
    df.loc[df['FamilySize'] == 1, 'IsAlone'] = 1
    return df
 
 
def fillna_most_frequent(df, column, column_new = None):
    """
    Gets the most frequent value of the column and replaces null-values with this value.
    """
 
    if column_new is None:
        column_new = column
 
    frequent = df[column].dropna().mode()[0]
    df[column_new] = df[column].fillna(frequent)
    return df
 
 
def fillna_median(df, column, column_new = None):
    """
    Fill null-values of a column with the median of the column.
    """
 
    if column_new is None:
        column_new = column
 
    median = df[column].dropna().median()
    df[column_new] = df[column].fillna(median)
    return df
 
 
def continuous_to_ordinal(df, groups, column, column_new = None):
    """
    Sorts continuous numerical value into groups.
    """
 
    if column_new is None:
        column_new = column
 
    df[column_new] = pd.qcut(df[column], groups, duplicates = 'drop')
    df = ordinal_to_numbers(df, column, column_new)
    return df
 
 
def ordinal_to_numbers(df, column, column_new = None):
    """
    Helper function to transform a column with ordinal values to a new column including a numeric index
    for the column.
    """
 
    if column_new is None:
        column_new = column
 
    values = df[column].unique()
    df[column_new] = df[column].map(lambda value: np.where(values == value)[0][0])
    return df
 
 
def prepare(df):
    df = extract_title(df)
    df = ordinal_to_numbers(df, 'Title')
    df = ordinal_to_numbers(df, 'Sex')
    df = fillna_most_frequent(df, 'Embarked')
    df = ordinal_to_numbers(df, 'Embarked')
    df = fillna_median(df, 'Fare')
    df = fillna_median(df, 'Age')
    df = continuous_to_ordinal(df, 4, 'Fare')
    df = continuous_to_ordinal(df, 5, 'Age')
    df = df.drop(['Name', 'Ticket', 'Cabin', 'PassengerId'], axis = 1)
    df = calculate_family_size(df)
 
    return df
 
 
train = prepare(train)
test = prepare(test)
 
train.to_pickle('./data/train_prepared.pkl')
test.to_pickle('./data/test_prepared.pkl')
```

Of course, in a real world scenario you would develop this file and run the program a few times until you know it has somehow the right capabilities. But when you know that the script is good enough to proceed, just add a new step to your DVC pipeline definition.

```bash
$ dvc run -d ./data/raw/train.csv -d ./data/raw/test.csv -f ./dvc/02-prepare-data.dvc -o ./data/train_prepared.pkl -o ./data/test_prepared.pkl python ./src/02-prepare.py
 
$ git add .
$ git commit -am "Added data preparation"
```

As above we again provided the files we depend on and the output files to DVC, as well as we defined to store the information about the step in `./dvc/02-prepare-data.dvc`. DVC can now build the dependency tree, thus when we change something in our first step (`01-fetch-data`) and run `dvc repro ./dvc/02-prepare-data.dvc` DVC will detect these changes and re-executes the first step before re-executing the 2nd.

Finally create a first baseline model. For this purpose we crate a new file `./src/03-training.py`:

```python
import pandas as pd
import numpy as np
import json
 
from sklearn.linear_model import LogisticRegression, RidgeClassifierCV
from sklearn.svm import SVC, LinearSVC
from sklearn.ensemble import (RandomForestClassifier, GradientBoostingClassifier)
from sklearn.neighbors import KNeighborsClassifier
from sklearn.naive_bayes import GaussianNB
from sklearn.linear_model import Perceptron
from sklearn.linear_model import SGDClassifier
from sklearn.tree import DecisionTreeClassifier
from sklearn.model_selection import cross_val_score, GridSearchCV
from sklearn.metrics import accuracy_score
 
# Result Dictionary
results = {}
 
# Split into x and y
train = pd.read_pickle('./data/train_prepared.pkl')
x_test = pd.read_pickle('./data/test_prepared.pkl')
x_train = train.drop(['Survived'], axis = 1)
y_train = train['Survived']
 
# Logistic Regression
logreg = LogisticRegression(solver='lbfgs', max_iter = 1000)
logreg.fit(x_train, y_train)
acc_log = round(logreg.score(x_train, y_train) * 100, 2)
results['Logistic Regression (ACC)'] = acc_log
 
# Support Vector Machines
svc = SVC(gamma='auto')
svc.fit(x_train, y_train)
acc_svc = round(svc.score(x_train, y_train) * 100, 2)
results['Support Vector Machines (ACC)'] = acc_svc
 
with open('./data/metrics.json', 'w') as fp:
    json.dump(results, fp, indent=2, sort_keys=True)
```

This training application uses some basic methods from SciKit-Learn to make predictions whether the Titanic passenger has survived or not. Let's add this final step to our DVC pipeline:

```bash
$ dvc run \
  -d ./src/03-training.py \
  -d ./data/train_prepared.pkl \
  -d ./data/test_prepared.pkl \
  -m ./data/metrics.json \
  -f ./Dvcfile \
  python ./src/03-training.py
 
$ git add .
$ git commit -am "Added baseline model"
$ git tag baseline
```

This last dvc run command has some deviations from the previous ones. First we also mentioned one file with the `-m` parameter. This indicates to DVC that this file contains metrics for our model. You'll see in a second how DVC uses this information. But basically it is just a usual output file, just with the information that it contains metrics.  Also we now called our output file `Dvcfile`. This is because this is the default filename when running `dvc repro` without any parameter. Thus anyone who checks out your project and wants to run the steps just needs to run dvc repro without any further knowledge.

From now on you can play witth any of the steps to improve your model. You may try other algorithms, refine the preparation steps, etc. To re-run the pipeline you always just execute dvc repro and DVC will detect what acxtually needs to be executed. For example, let's add another algorithm to the `./src/03-training.py` - File:

```python
# Decision Tree
decision_tree = DecisionTreeClassifier()
decision_tree.fit(x_train, y_train)
acc_decision_tree = round(decision_tree.score(x_train, y_train) * 100, 2)
results['Decission Tree (ACC)'] = acc_decision_tree
```

When you run this with dvc repro, it will only execute the training step again:

```
$ dvc repro
WARNING: assuming default target 'Dvcfile'.
Stage 'dvc/01-fetch-data.dvc' didn't change.
Stage 'dvc/02-prepare-data.dvc' didn't change.
WARNING: Dependency 'src/03-training.py' of 'Dvcfile' changed because it is 'modified'.
WARNING: Stage 'Dvcfile' changed.
Running command:
    python ./src/03-training.py
Saving information to 'Dvcfile'.
 
To track the changes with git, run:
 
    git add Dvcfile
```

Let's discover the metrics feature of DVC. As we have now trained the model two times with small changes, we might be interested in comaring their outcomes. We can easily do that with `dvc metrics show`:

```
$ dvc metrics show -a -T
working tree:
    data/metrics.json:
        {
          "Logistic Regression (ACC)": 80.13,
          "Support Vector Machines (ACC)": 83.28
        }
baseline:
    data/metrics.json:
        {
          "Logistic Regression (ACC)": 80.13,
          "Support Vector Machines (ACC)": 83.28
        }
```

This will provide as with the contents of all metric files of the existing tags in our Git Project and allows us to compare our different experiments.

In the previous steps we have seen how tags can help us to develop and fine-grain our model. But if we want to compare totally different approaches, Git tags might not be the right way to work with. Because you might want to go-back to your previous approach w/o loosing the information about the tests you have done with your primary approach. In such a case it's a good idea to create a new Gir branch. Let's try that in our project. Create a new branch:

```bash
$ git add .
$ git commit -am "Added decision tree"
$ git tag "with-decission-tree"
$ git checkout -b tensorflow
```

Now replace the content of ./src/03-training.py with a Tensorflow training program:

```python
import pandas as pd
import json
import tensorflow as tf
 
from collections import namedtuple
from sklearn.preprocessing import LabelBinarizer
from sklearn.model_selection import train_test_split
 
 
def split_valid_test_data(data, fraction=(1 - 0.8)):
    data_y = data["Survived"]
    lb = LabelBinarizer()
    data_y = lb.fit_transform(data_y)
 
    data_x = data.drop(["Survived"], axis=1)
 
    train_x, valid_x, train_y, valid_y = train_test_split(data_x, data_y, test_size=fraction)
 
    return train_x.values, train_y, valid_x, valid_y
 
 
def build_neural_network(train_x, hidden_units=10):
    tf.reset_default_graph()
    inputs = tf.placeholder(tf.float32, shape=[None, train_x.shape[1]])
    labels = tf.placeholder(tf.float32, shape=[None, 1])
    learning_rate = tf.placeholder(tf.float32)
    is_training=tf.Variable(True,dtype=tf.bool)
 
    initializer = tf.contrib.layers.xavier_initializer()
    fc = tf.layers.dense(inputs, hidden_units, activation=None,kernel_initializer=initializer)
    fc=tf.layers.batch_normalization(fc, training=is_training)
    fc=tf.nn.relu(fc)
 
    logits = tf.layers.dense(fc, 1, activation=None)
    cross_entropy = tf.nn.sigmoid_cross_entropy_with_logits(labels=labels, logits=logits)
    cost = tf.reduce_mean(cross_entropy)
 
    with tf.control_dependencies(tf.get_collection(tf.GraphKeys.UPDATE_OPS)):
        optimizer = tf.train.AdamOptimizer(learning_rate=learning_rate).minimize(cost)
 
    predicted = tf.nn.sigmoid(logits)
    correct_pred = tf.equal(tf.round(predicted), labels)
    accuracy = tf.reduce_mean(tf.cast(correct_pred, tf.float32))
 
    # Export the nodes
    export_nodes = ['inputs', 'labels', 'learning_rate','is_training', 'logits',
                    'cost', 'optimizer', 'predicted', 'accuracy']
    Graph = namedtuple('Graph', export_nodes)
    local_dict = locals()
    graph = Graph(*[local_dict[each] for each in export_nodes])
 
    return graph
 
 
def get_batch(data_x,data_y,batch_size=32):
    batch_n=len(data_x)//batch_size
    for i in range(batch_n):
        batch_x=data_x[i*batch_size:(i+1)*batch_size]
        batch_y=data_y[i*batch_size:(i+1)*batch_size]
 
        yield batch_x,batch_y
 
 
# Result Dictionary
results = {
    'NN (ACC)': 0
}
 
# Split into x and y
train = pd.read_pickle('./data/train_prepared.pkl')
x_test = pd.read_pickle('./data/test_prepared.pkl')
 
x_train, y_train, x_valid, y_valid = split_valid_test_data(train)
model = build_neural_network(x_train)
 
epochs = 200
train_collect = 50
train_print=train_collect*2
 
learning_rate_value = 0.001
batch_size=16
 
x_collect = []
train_loss_collect = []
train_acc_collect = []
valid_loss_collect = []
valid_acc_collect = []
 
saver = tf.train.Saver()
with tf.Session() as sess:
    sess.run(tf.global_variables_initializer())
    iteration=0
    for e in range(epochs):
        for batch_x,batch_y in get_batch(x_train,y_train,batch_size):
            iteration+=1
            feed = {model.inputs: x_train,
                    model.labels: y_train,
                    model.learning_rate: learning_rate_value,
                    model.is_training:True
                    }
 
            train_loss, _, train_acc = sess.run([model.cost, model.optimizer, model.accuracy], feed_dict=feed)
 
            if iteration % train_collect == 0:
                x_collect.append(e)
                train_loss_collect.append(train_loss)
                train_acc_collect.append(train_acc)
 
                if iteration % train_print==0:
                    print("Epoch: {}/{}".format(e + 1, epochs),
                          "Train Loss: {:.4f}".format(train_loss),
                          "Train Acc: {:.4f}".format(train_acc))
 
                feed = {model.inputs: x_valid,
                        model.labels: y_valid,
                        model.is_training:False
                        }
                val_loss, val_acc = sess.run([model.cost, model.accuracy], feed_dict=feed)
                valid_loss_collect.append(val_loss)
                valid_acc_collect.append(val_acc)
 
                if iteration % train_print==0:
                    print("Epoch: {}/{}".format(e + 1, epochs),
                          "Validation Loss: {:.4f}".format(val_loss),
                          "Validation Acc: {:.4f}".format(val_acc))
 
                    if results['NN (ACC)'] < val_acc:
                        results['NN (ACC)'] = val_acc
 
 
# saver.save(sess, "../data/titanic.ckpt")
 
results = {
    'NN (ACC)': str(results['NN (ACC)'])
}
 
with open('./data/metrics.json', 'w') as fp:
    json.dump(results, fp, indent=2, sort_keys=True)
```

Again just execute dvc repro to train the model with Tensorflow:

```bash
$ dvc repro
[...]
Epoch: 198/200 Validation Loss: 0.6101 Validation Acc: 0.8156
Epoch: 200/200 Train Loss: 0.3136 Train Acc: 0.8694
Epoch: 200/200 Validation Loss: 0.6118 Validation Acc: 0.8156
Saving 'data/metrics.json' to '.dvc/cache/8d/24923f0e0b18308fbd48ae5de0bb99'.
Saving information to 'Dvcfile'.
 
To track the changes with git, run:
 
    git add Dvcfile
```

Again, DVC detects that only the training application needs to be run again, it runs it and stores the Metrics. And of course you can compare it to the other metrics:

```bash
$ dvc metrics show -a -T
working tree:
    data/metrics.json:
        {
          "NN (ACC)": "0.8435754"
        }
master:
    data/metrics.json:
        {
          "Decission Tree (ACC)": 89.56,
          "Logistic Regression (ACC)": 80.13,
          "Support Vector Machines (ACC)": 83.28
        }
tensorflow:
    data/metrics.json:
        {
          "NN (ACC)": "0.8435754"
        }
baseline:
    data/metrics.json:
        {
          "Logistic Regression (ACC)": 80.13,
          "Support Vector Machines (ACC)": 83.28
        }
```

You may to decide to improve your Tensorflow model, but maybe at some point you want to go back to your first approach and want to improve this one. In that case you can simply switch with Git and DVC checkout:

```
$ git checkout
$ dvc checkout
``` 

As mentioned above, `dvc checkout` will checkout the data files which belong to your commit (all data-files, including the metrics are not stored in Git)

# Summary and Conclusion

As seen in the steps above DVC can help you to cache intermediate and final results of your model training process. On top of that it comes with a handy possibiloty to track and compare metrics of different experiments. In combination with the data repositories like Kaggle it allows you to clearly specify the data and steps required to reproduce your results. As such DVC, Git and a Data Repository are the foundation for reproducable data science products.

In combination with Git best practices for using Tags for minor progress in your development or Branches for completely different experiments, DVC becomes a powerful tool which can be used during exploratioon, analysis and further during development of the actual model.