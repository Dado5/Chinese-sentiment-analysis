# Chinese-sentiment-analysis
 "from sklearn.cross_validation import train_test_split\n",
    "from gensim.models.word2vec import Word2Vec\n",
    "import numpy as np\n",
    "import pandas as pd\n",
    "import jieba\n",
    "from sklearn.externals import joblib\n",
    "from sklearn.svm import SVC\n",
    "import sys  \n",
    "reload(sys)  \n",
    "sys.setdefaultencoding('utf8')"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### 载入数据，做预处理(分词)，切分训练集与测试集"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {
    "collapsed": true
   },
   "outputs": [],
   "source": [
    "def load_file_and_preprocessing():\n",
    "    neg=pd.read_excel('data/neg.xls',header=None,index=None)\n",
    "    pos=pd.read_excel('data/pos.xls',header=None,index=None)\n",
    "\n",
    "    cw = lambda x: list(jieba.cut(x))\n",
    "    pos['words'] = pos[0].apply(cw)\n",
    "    neg['words'] = neg[0].apply(cw)\n",
    "\n",
    "    #print pos['words']\n",
    "    #use 1 for positive sentiment, 0 for negative\n",
    "    y = np.concatenate((np.ones(len(pos)), np.zeros(len(neg))))\n",
    "\n",
    "    x_train, x_test, y_train, y_test = train_test_split(np.concatenate((pos['words'], neg['words'])), y, test_size=0.2)\n",
    "    \n",
    "    np.save('svm_data/y_train.npy',y_train)\n",
    "    np.save('svm_data/y_test.npy',y_test)\n",
    "    return x_train,x_test"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### 对每个句子的所有词向量取均值，来生成一个句子的vector"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {
    "collapsed": true
   },
   "outputs": [],
   "source": [
    "def build_sentence_vector(text, size,imdb_w2v):\n",
    "    vec = np.zeros(size).reshape((1, size))\n",
    "    count = 0.\n",
    "    for word in text:\n",
    "        try:\n",
    "            vec += imdb_w2v[word].reshape((1, size))\n",
    "            count += 1.\n",
    "        except KeyError:\n",
    "            continue\n",
    "    if count != 0:\n",
    "        vec /= count\n",
    "    return vec"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### 计算词向量"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {
    "collapsed": true
   },
   "outputs": [],
   "source": [
    "#计算词向量\n",
    "def get_train_vecs(x_train,x_test):\n",
    "    n_dim = 300\n",
    "    #初始化模型和词表\n",
    "    imdb_w2v = Word2Vec(size=n_dim, min_count=10)\n",
    "    imdb_w2v.build_vocab(x_train)\n",
    "    \n",
    "    #在评论训练集上建模(可能会花费几分钟)\n",
    "    imdb_w2v.train(x_train)\n",
    "    \n",
    "    train_vecs = np.concatenate([build_sentence_vector(z, n_dim,imdb_w2v) for z in x_train])\n",
    "    #train_vecs = scale(train_vecs)\n",
    "    \n",
    "    np.save('svm_data/train_vecs.npy',train_vecs)\n",
    "    print train_vecs.shape\n",
    "    #在测试集上训练\n",
    "    imdb_w2v.train(x_test)\n",
    "    imdb_w2v.save('svm_data/w2v_model/w2v_model.pkl')\n",
    "    #Build test tweet vectors then scale\n",
    "    test_vecs = np.concatenate([build_sentence_vector(z, n_dim,imdb_w2v) for z in x_test])\n",
    "    #test_vecs = scale(test_vecs)\n",
    "    np.save('svm_data/test_vecs.npy',test_vecs)\n",
    "    print test_vecs.shape"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {
    "collapsed": true
   },
   "outputs": [],
   "source": [
    "def get_data():\n",
    "    train_vecs=np.load('svm_data/train_vecs.npy')\n",
    "    y_train=np.load('svm_data/y_train.npy')\n",
    "    test_vecs=np.load('svm_data/test_vecs.npy')\n",
    "    y_test=np.load('svm_data/y_test.npy') \n",
    "    return train_vecs,y_train,test_vecs,y_test"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### 训练svm模型"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {
    "collapsed": true
   },
   "outputs": [],
   "source": [
    "def svm_train(train_vecs,y_train,test_vecs,y_test):\n",
    "    clf=SVC(kernel='rbf',verbose=True)\n",
    "    clf.fit(train_vecs,y_train)\n",
    "    joblib.dump(clf, 'svm_data/svm_model/model.pkl')\n",
    "    print clf.score(test_vecs,y_test)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### 构建待预测句子的向量"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {
    "collapsed": true
   },
   "outputs": [],
   "source": [
    "def get_predict_vecs(words):\n",
    "    n_dim = 300\n",
    "    imdb_w2v = Word2Vec.load('svm_data/w2v_model/w2v_model.pkl')\n",
    "    #imdb_w2v.train(words)\n",
    "    train_vecs = build_sentence_vector(words, n_dim,imdb_w2v)\n",
    "    #print train_vecs.shape\n",
    "    return train_vecs"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### 对单个句子进行情感判断 "
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {
    "collapsed": true
   },
   "outputs": [],
   "source": [
    "def svm_predict(string):\n",
    "    words=jieba.lcut(string)\n",
    "    words_vecs=get_predict_vecs(words)\n",
    "    clf=joblib.load('svm_data/svm_model/model.pkl')\n",
    "     \n",
    "    result=clf.predict(words_vecs)\n",
    "    \n",
    "    if int(result[0])==1:\n",
    "        print string,' positive'\n",
    "    else:\n",
    "        print string,' negative'"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {
    "collapsed": true
   },
   "outputs": [],
   "source": [
    "##对输入句子情感进行判断\n",
    "string='电池充完了电连手机都打不开.简直烂的要命.真是金玉其外,败絮其中!连5号电池都不如'\n",
    "#string='牛逼的手机，从3米高的地方摔下去都没坏，质量非常好'    \n",
    "svm_predict(string)"
   ]
  }
 ],
 "metadata": {
  "kernelspec": {
   "display_name": "Python 2",
   "language": "python",
   "name": "python2"
  },
  "language_info": {
   "codemirror_mode": {
    "name": "ipython",
    "version": 2
   },
   "file_extension": ".py",
   "mimetype": "text/x-python",
   "name": "python",
   "nbconvert_exporter": "python",
   "pygments_lexer": "ipython2",
   "version": "2.7.10"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 0
}
