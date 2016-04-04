# MaxEnt source code review

Source code review of Tsuruoka's C++ library for maximum entropy classification (see [http://www.nactem.ac.uk/tsuruoka/maxent/](http://www.nactem.ac.uk/tsuruoka/maxent/)).


## Introduction

On his website, [Tsuruoka](http://www.logos.ic.i.u-tokyo.ac.jp/~tsuruoka/) proposed a C++ library for maximum entropy classification. In order to get a better and deeper understanding of implementation details, I propose here a simple code review. The code base is relatively small (around 2500 lines of code). Those notes are primary destined for my personal use and reflect my current understanding. I propose them here, in case where that could help someone.

This article is divided in three parts:

 - The core
 - Optimization methods
 - Postagging example

But, before digging ahead, let's get an overview of the code base.


### Overview

Using [Rexdep](https://github.com/itchyny/rexdep), we can obtain the following directed graph of dependencies:

![Dependencies](https://raw.githubusercontent.com/fauconnier/code_review_tsuruoka/master/resources/dependencies.png)

The __core of the library__ are the header `maxent.h`, and its implementation `maxent.cpp`. In this core, the fundamental structures and functions of the learning task are defined (training set struct, functions for computing of objective, etc.). The following section  will discuss some of these aspects.

Besides that, three __optimization methods__ are proposed:

 - `lbfgs.cpp`: the very efficient [BFGS algorithm](https://en.wikipedia.org/wiki/Broyden–Fletcher–Goldfarb–Shanno_algorithm) in limited memory version;
 - `owlqn.cpp`: a [variant of BFGS](https://en.wikipedia.org/wiki/Limited-memory_BFGS#OWL-QN) for L1-regularized models;
 - `sgd.cpp`: the classic [stochastic gradient descent](https://en.wikipedia.org/wiki/Stochastic_gradient_descent);

Note that, due to their shared nature, `lbfgs.cpp` and `owlqn.cpp` include both the header `lbfgs.h`.

Two __classification examples__ are proposed:

 - `bicycle.cpp`: binary classification between cars and bicycles given simple features. This example is made to just figure out the basic principles;
 - `postaggin.cpp`: the classic task which aims at identifying the right part-of-speech (POS) for a given token. A little dataset of 200 sentences is given to train and test new models;


### A word about `mathvec.h`

The `mathvec.h` describes the class `Vec` which extends the possibilities of `vector<double>`:  the operators (e.g. `<<`, binary `+`, etc.) are overloaded and some facilities are in place for computing the dot product and the vector projection. Those elements are used in `lbfgs.cpp`, `owlqn.cpp` and `maxent.h`.

In this case, the optimization aspect is taken into account. For instance, the dot production function is [inlined](https://en.wikipedia.org/wiki/Inline_function), in order to save CPU registers and increase the efficience of operation.

```C++
inline double dot_product(const Vec & a, const Vec & b)
{
  double sum = 0;
  for (size_t i = 0; i < a.Size(); i++) {
    sum += a[i] * b[i];
  }
  return sum;
}
```



## The core

The class `ME_Model` is the core of the library. This is reflected in the space  that class takes in `maxent.h` (about 80% of the written source code). A couple of nested structures are defined inside `ME_Model`. Note that nesting does not imply other relations than a shared scope for the nested structures. Inheritance or composition are not involved, unless they are explicitly stated. The following figure gives an insight about the global architecture:

![maxent.h](./resources/maxent_h.png)


### `ME_Sample` and  `Sample`

The structure `ME_Sample` records the observations. Each `ME_Sample` have its  own features. There are two kinds of features: binary (`vector<string>`) and real-valued (`vector<pair<string, double>>`) features. 

When the `ME_Sample` records are added to the training set (`void add_training_sample(const ME_Sample & mes)`), they are transformed in `Sample`: a distinction is made here between the _external observations_ (`ME_Sample`) and the _internal representation of those observations_ (`Sample`), inaccessible outside the `ME_Model`.

During this transformation, since it's easier to handle integers in place of strings, then a common trick is to map each string label (and each features) to an integer identifier using an hash map:

![hash_table](./resources/hash_table.png)


### Hash implementation

By default, pre-processor instructions in `maxent.h` call the header `ext/hash_map`. This implementation of hash table was [one the first largely used](https://en.wikipedia.org/wiki/Unordered_associative_containers_%28C%2B%2B%29#History), with `hash_set`, `hash_multimap` and `hash_multiset`. However, this is now outdated and kept  only for downgraded compatibility. Thus, compiling the project using this kind of hash lead to a warning with a C++11 compiler. As indicated in the source, the best way to skip the warning is to comment the macro which defines `USE_HASH_MAP`.

```C++
#define USE_HASH_MAP  // if you encounter errors with hash, try commenting out this line.  (the program will be a bit slower, though)
#ifdef USE_HASH_MAP
#include <ext/hash_map>
#endif
```

As the comments indicate,  the program will be a bit slower. This is expected since the standard library container `map` orders its keys. A good alternative would be to use an [unordered_map](https://en.wikipedia.org/wiki/Unordered_associative_containers_%28C%2B%2B%29) (maybe a pull request?). The C++ Technical Report 1 ([TR1](https://en.wikipedia.org/wiki/C%2B%2B_Technical_Report_1)), whose improvements contributed to C++11, suggests that way. If you are interested, check out this article: [Hash Table Performance Tests](http://preshing.com/20110603/hash-table-performance-tests/).


### hashfun_str

The structure `hashfun_str` is used for the definition of  `ext/hash_map` into the scope of `MiniStringBag`.

```C++
#ifdef USE_HASH_MAP
    typedef __gnu_cxx::hash_map<std::string, int, hashfun_str> map_type;
#else    
    typedef std::map<std::string, int> map_type;
#endif
```

By default, the `ext/hash_map` implementation cannot deal with string keys. This is partly because this hash_map is not a part of the standard library. Thus, in order to records labels and features as keys, it is necessary to give a hash function to handle the string. This is the goal of the `hashfun_str` structure which is an function object. As we can read into (Stroupstrup, 1997):

> An object of a class with an application operator (§11.9) is called a _function-like object_, a _functor_ or simply a _function object_.
(18.4, Stroupstrup, 1997)


```C++
  struct hashfun_str
  {
    size_t operator()(const std::string& s) const {
      assert(sizeof(int) == 4 && sizeof(char) == 1);
      const int* p = reinterpret_cast<const int*>(s.c_str());
      size_t v = 0;
      int n = s.size() / 4;
      for (int i = 0; i < n; i++, p++) {
        //      v ^= *p;
        v ^= *p << (4 * (i % 2)); // note) 0 <= char < 128
      }
      int m = s.size() % 4;
      for (int i = 0; i < m; i++) {
        v ^= s[4 * n + i] << (i * 8);
      }
      return v;
    }
  };
```



The hash function proposed in this structure uses extensively low-level bitwise operations to generate the hash value. Using this it's possible to obtain significant speed improvements (see benchmark directory).



### Adding a sample to the training set

The function `add_training_sample(const ME_Sample & mes)`


```C++
void ME_Model::add_training_sample(const ME_Sample & mes) {

    Sample s;
    s.label = _label_bag.Put(mes.label);
    if (s.label > ME_Feature::MAX_LABEL_TYPES) {
        cerr << "error: too many types of labels." << endl;
        exit(1);
    }
    for (vector<string>::const_iterator j = mes.features.begin(); j != mes.features.end(); j++) {
        s.positive_features.push_back(_featurename_bag.Put(*j));
    }
    for (vector<pair<string, double> >::const_iterator j = mes.rvfeatures.begin(); j != mes.rvfeatures.end(); j++) {
        s.rvfeatures.push_back(pair<int, double>(_featurename_bag.Put(j->first), j->second));
    }
    std::cout << "ref_model " <<  _ref_modelp << std::endl;
    if (_ref_modelp != NULL) {
        ME_Sample tmp = mes;; // Typo: double ';'
        s.ref_pd = _ref_modelp->classify(tmp); // Why do that?
    }
    std::cout << s.label << " ref_pd:" << s.ref_pd.size() << std::endl;
    //  cout << s.label << "\t";
    //  for (vector<int>::const_iterator j = s.positive_features.begin(); j != s.positive_features.end(); j++){
    //    cout << *j << " ";
    //  }
    //  cout << endl;

    _vs.push_back(s);
}
```


### ME_Feature

The structure `ME_Feature` 


```C++
  struct ME_Feature
  {
    enum { MAX_LABEL_TYPES = 255 };
      
    //    ME_Feature(const int l, const int f) : _body((l << 24) + f) {
    //      assert(l >= 0 && l < 256);
    //      assert(f >= 0 && f <= 0xffffff);
    //    };
    //    int label() const { return _body >> 24; }
    //    int feature() const { return _body & 0xffffff; }
    ME_Feature(const int l, const int f) : _body((f << 8) + l) {
      assert(l >= 0 && l <= MAX_LABEL_TYPES);
      assert(f >= 0 && f <= 0xffffff);
    };
    int label() const { return _body & 0xff; }
    int feature() const { return _body >> 8; } // bitwise operator
    // Is like to make some unsigned int /= 8
    unsigned int body() const { return _body; }
  private:
    unsigned int _body;
  };
```




### FeatureBag

Two data structures are involved in `ME_FeatureBag`:

 - Hash implementation with pairs `<unsigned int, int>` called `mef2id`
 
 - Standard vector of `ME_Features`, called  `id2mef`.

Note that `mef` probably stands for `ME_Feature`. 

Two functions are defined for FeatureBag:

 - The put function:

```C++
int Put(const ME_Feature & i) {
  map_type::const_iterator j = mef2id.find(i.body());
  if (j == mef2id.end()) {
    int id = id2mef.size();
    id2mef.push_back(i);
    mef2id[i.body()] = id;
    return id;
  }
  return j->second;
}
```



## Optimization methods

By default, this library use 'LBFGS' to optimize the objective.

### LBFGS





## Example of postagging


## References


 - Stroustrup, B. (1997). The C++ Programming Language. Third edition. Pearson Education India.


