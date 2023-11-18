- [ ] Port [Data Structures & Algorithms in Dart](https://www.kodeco.com/books/data-structures-algorithms-in-dart/v2.0) to Pharo
  - [ ] Compare with [Pharo Containers](https://github.com/pharo-containers)
- [ ] Check implementation of Parameterized Tests
- [ ] FFI
    - [ ] libclang
        - [ ] First use Dart's FFI to see what will be generated from [Index.h](https://github.com/llvm/llvm-project/blob/main/clang/include/clang-c/Index.h)
            - [libclang Tutorial](https://clang.llvm.org/docs/LibClang.html)
        - [ ] FDB
        - [ ] raylib
        - [ ] Skia
- [x] Book: [Pharo by Example](https://github.com/SquareBracketAssociates/NewPharoByExample9/releases/download/latest/PharoByExample9-wip.pdf)
    - [ ] Debug (p. 287): What is (method parseTree ...)
          ((RBBrowserEnvironment new forClasses: (Collection withAllSubclasses))
            selectMethods: [:method |
            method sendsToSuper
            and: [(method parseTree superMessages includes: method selector)
            not]])
- [x] Word count: https://gist.github.com/mkohlhaas/bf50d06c1646eb1fa465dd4aeaf06731
