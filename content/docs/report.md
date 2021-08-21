---
title: "Final Report"
date: 2021-08-21T09:15:00+05:30
draft: false
---

# Making Smart Pointer Checkers default checkers in the Static Analyzer

<h2 style="text-align: center; color: #a2a6ab">Final Report: Google Summer of Code 2021</h2>

<h4 style="text-align: center">Deep Majumder</h4>
<h4 style="text-align: center">deep.majumder2019@gmail.com</h4>

## Overview 
This project aims to complete the `SmartPtrChecker` and thus `SmartPtrModeling` checkers to detect null-dereferences of the `std::unique_ptr`. This is a continuation of the GSoC 2020 [project](https://docs.google.com/document/d/1WZSt45kZUhg0UbOv0HXBhyEYaHrb-G-TpEhj_nU041Q/edit) in the same area.

## Description
`SmartPtrChecker` used to be a checker in the Clang Static Analyzer for detecting "simple" null dereferences. The GSoC 2020 project improved upon this to provide the `SmartPtrModeling` checker, which models many of the important operations on `std::unique_ptr`, such as move, reset, release, convert to bool, etc. This project completes most of the gaps and uncompleted areas in the modelling, including the ubiquitous `std::make_unique` function and the destructor (which communicates with the `MallocChecker` to remove false positives). *Currently, we are a few bug fixes and some polishing away from making this a default checker* (as of Clang 13, it is an alpha checker).

## Revisions

One aim of this project was to make sure that no method defined on `std::unique_ptr` is left un-modelled. If we don't, CSA either does:

- *Conservative evaluation*, which leads to loss of information and possibly warnings being suppressed
- *Inlining*, which sometimes leads the CSA to falsely believe that there is a bug in the standard library code (because we have modelled parts of it and are inlining other parts). Since bugs in standard library code are normally suppressed, this leads to other bugs beings suppressed (false-negatives). For example:

```cpp
void foo() {
    auto smart = std::unique_ptr<int>(new int(13));
    auto raw = new int(29);
    // There is a leak here.
}
```
Without [D105821](https://reviews.llvm.org/D105821), this used to have **no** leak warning. The inlining of the destructor of `std::unique_ptr` led the CSA to falsely believe there was a bug in the destructor code.

### Model comparision methods of `std::unique_ptr`

[D104616](https://reviews.llvm.org/D104616) introduces modelling of comparision operator overloads of `std::unique_ptr`. It leverages the `SValBuilder::evalBinOp` to evaluate the result of the operator. More importantly, it attempts to perform a **state-split** if it is possible.

### Model `std::swap` specialization for `std::unique_ptr`

[D104300](https://reviews.llvm.org/D104300) models the `std::swap` specialization for `std::unique_ptr`. There is an existing `swap` method on `std::unique_ptr`, which performs roughly the same thing. Thus, the common code is refactored out and both methods are handled exactly the same way.

![std-swap](assets/std-swap.png)


### Model `<<` operator specialization for `std::unique_ptr`

[D105421](https://reviews.llvm.org/D105421) models the `<<` operator specialization for `std::unique_ptr`. There is not that much modelling required for this method, other than invalidating the stream region.


### Model `std::make_unique` and cousins

[D103750](https://reviews.llvm.org/D103750) models the quintessential `std::make_unique` function. In this entire checker, we only account for `unique_ptr` containing pointers, not arrays - we do the same here as well. The crux of this patch is informing the CSA that we are constructing an object via this function. Ideally this should be handled automatically (as it is for constructors). But here we bail out and simply call `ExprEngine::updateObjectsUnderConstruction`. Also we ensure that the `ProgramState` knows that we have a non-null inner pointer in the `unique_ptr`.


### Fix for faulty namespace test and add flag checking

[D106296](https://reviews.llvm.org/D106296) prevents a crash we encountered by handling the case where we do not have the declaration of a function and so cannot know its namespace. Also we add the `ModelSmartPtrDereference` flag check to the previously modelled functions.


### Add option to `SATest.py` for extra checkers

[D106739](https://reviews.llvm.org/D106739) augments the functionality of the `SATest.py` script. This script runs the CSA on the projects in the `clang/utils/analyzer/projects` directory (via a Docker image). By default, the script runs the CSA with only default checkers enabled. This patch adds a flag to enable extra checkers (for our case, the `SmartPtrChecker`), enabling us to conveniently run test our patches.


### Model destructor for `std::unique_ptr`

[D105821](https://reviews.llvm.org/D105821) models the elephant in the room - the destructor for `std::unique_ptr`. Although we are just modelling the destructor in this patch, in reality, we have ended up calling the `MallocChecker` and improving the invalidation scheme in addition to the modelling. We are doing the following steps:

- Enable calling `evalCall` for destructor-calls (before this patch, only the `defaultEvalCall` is run)
- Escape inner pointer on construction and reset
- Escape all reachable symbols in `checkRegionChange`
- *Invalidate* the inner pointer and remove it from `TrackedRegionMap` (in both destructor and in reset)

The fuss about *invalidation* of inner pointer stems from the fact that `delete` also calls the *member destructor* (if it exists). Consider the following class:

```cpp
class LameVector {
    size_t sz = 0;
    size_t cap = 4;
    size_t el_sz = 1;
    unsigned char *buf;

public:
    LameVector() : buf{new unsigned char[cap]} {}
    ~LameVector() { delete[] buf; }
};

void foo() {
    auto ptr = std::make_unique<LameVector>();
}
```

In this code, `LameVector` contains an inner pointer, which is freed in its destructor. If we do not invalidate this inner pointer, CSA will think `LameVector` and hence `ptr` leaks memory.

This is a stop gap measure and it seems to work for the time being. The proper solution is to inline both the member constructor (both during constructor calls and during `make_unique`) and the member destructor. But we don't yet have a mechanism to "evaluate" functions from a checker.

## Results

We have two sources of test projects to run:- [projects](https://github.com/llvm/llvm-project/tree/main/clang/utils/analyzer/projects) in the `clang/utils/analyzer/projects` directory and [Webkit](https://github.com/WebKit/WebKit). We evaluated checker performance by:

- Running scan-build with and without the patches from this project
- Running scan-build with `SmartPtrChecker` enabled and disabled

### Projects in `clang/utils/analyzer/projects`

- On running with and without the patches of this project, **no warnings were added or removed**. This is somewhat expected since the main false warnings expected to be fixed are from the `MallocChecker`. This effect is cancelled by the suppressed error detected during inlining of `std::unique_ptr` destructor.

- On running with `SmartPtrChecker` enabled and disabled, **some warnings were added, many were removed**. The warnings removed were almost all false memory-leak warnings due to the lack of modelling. (`drogon`, `fmt`, `re2`, `faiss`). `oatpp` had some "C++ move semantics"  warnings removed, because `std::unique_ptr` or `srd::shared_ptr` is perfectly usable and valid after a move. Some warnings were added to `duckdb`, `drogon` and `re2`, all due to custom "smart pointers" in the codebase.

### Webkit

- On running with and without the patches of this project, about **250** false memory leaks warnings were removed. As for null-dereference warnings, **6** were removed and **6** added. The ones removed is a question for further investigation.

- On running with `SmartPtrChecker` enabled and disabled, about **90** null smart pointer dereference warnings are emitted (including 3 out the 4 mentioned in previous year's report, the fourth was not emitted due to the code being removed). *There are still some false positives*. These will be areas to look at before declaring that we have a stable checker.

![WebkitBug](assets/new-webkit.png)

## Future Work

- **Inlined defensive checks**: This is a class of false-positives with a really misleading note.

```cpp
void foo(std::unique_ptr<int> &P) {
    if (P) {}
}

int bar(std::unique_ptr<int> Q) {
    foo(Q);
    return *Q; // warning: null dereference
}
```
The problem is that `foo(Q)` creates a state split. But CSA uses the state split in `foo` to conclude in `bar()` that there exists a path in which `Q` is null and there is a dereference. We need to make sure that state-splits in other functions don't lead to misleading diagnostics.

- **Polish and commit [D105821](https://reviews.llvm.org/D105821)**: This patch has become too bulky and probably needs to be split into two at least (one part for the destructor and the other part for the pointer escape). Also they need *tests*!

- **Enable the checker by default**: Once the previous two tasks are done and the remaining bugs uncovered by WebKit fixed, we can make it a default checker. 😃

- **Model `std::shared_ptr` and `std::weak_ptr`**: These two can be modelled in a manner similar to `std::unique_ptr`, ie, with the `TrackedRegionMap`. In addition, these smart pointers need a *ref count* to be stored in the GDM. A discussion on how to do this can be found in my GSoC [proposal](https://docs.google.com/document/d/1DlU7Whg33qAp3wHBdAcGvl8VE2gfslaGBc-GRb_CNjk/edit?usp=sharing).

## Running the code

To run the CSA with `SmartPtrChecker` enabled:
```shell
<path-to-scan-build> -o . -enable-checker alpha.cplusplus.SmartPtr -analyzer-config cplusplus.SmartPtrModeling:ModelSmartPtrDereference=true clang++ -c <filename>
```

## Acknowledgement

I would like to express my sincere gratitude towards my four mentors - **Artem Dergachev**, **Valeriy Savchenko**, **Gábor Horváth** and **Raphael Isemann**. They helped me tirelessly throughout the GSoC period, quickly responding to my emails, reviewing my patches and giving advice. They were available for long hours of Skype calls, at often awkward times for them (due to time zone differences). It was a fun three months of learning, coding, struggling, hacking and doing something, perhaps, meaningful.