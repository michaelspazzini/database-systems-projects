# create minimal env
conda create -n sistemi-evolutivi-big-data python=3.11 -y
conda activate sistemi-evolutivi-big-data

# use conda-forge
conda config --env --add channels conda-forge
conda config --env --set channel_priority strict

# install package
conda install -y ipykernel pymongo scikit-learn pandas nltk
python -m pip install --upgrade pip

# registration kernel for Jupyter/VS Code
python -m ipykernel install --user --name sebd --display-name "Python (conda sis-evo-big-data)"