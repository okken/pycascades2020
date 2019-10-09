class: center, middle

# Multiply your Testing Effectiveness with Parameterized Testing

Brian Okken

[@brianokken](https://twitter.com/brianokken)

Code and markdown for slides
[github.com/okken/presentation_parametrization](https://github.com/okken/presentation_parametrization)
---


# Brian Okken

.left-column[
Work

<img src="images/r_s_logo.png" width="300">

Podcasts

<a href="https://pythonbytes.fm">
<img src="images/pb.png" width="180"></a>
<a href="https://testandcode.com">
<img src="images/testandcode.jpg" width="130"></a>
]
.right-column[
Book
 
<a href="https://t.co/AKfVKcdDoy?amp=1">
<img src="images/book.jpg" style="border-style: solid;" width="200">
</a>
]

---

# Outline

* Value of Tests
* Parametrize vs Parameterize
* Examples:
    * without Parametrization
    * Function Parametrization
    * Fixture Parametrization
    * `pytest_generate_tests()`
* Choosing a Technique
* Combining Techniques
* Resources

---

# Value of Tests

Automated tests give us confidence.
What does confidence look like?

???
* We all know that tests are important.
* Imagine a CI pipeline with
    * a full suite of tests 
    * running against every merge request 
    * before code review
* We want these tests to give us confidence in software under test


--

A passing test suite means:

* I didn't break anything that used to work.
* Future changes won’t break current features.
* The code is ready for users.
* I can refactor until I'm proud of the code.
* Code reviews can focus on team understanding and ownership.

Only works if:

* New features are tested with new tests.
* Tests are easy and fast to write.
---

# Parametrize vs Parameterize

**parameter** + **ize**

* paramet_erize_ (US)
* paramet_rize_ (UK)

`pytest` uses `parametrize`, the UK spelling.

I've tried to get them to change it.  
They don't want to.  
I've gotten over it. 
???
* We're going to use parametrization to write more test cases with fewer test functions.
* This also makes tests easier to maintain.
* pytest uses the UK spelling of parametrize. 
* No e between the t and the r.
---

# Something to Test

`triangles.py`:
```python
def triangle_type(a, b, c):
    """
    Given three angles,
    return 'obtuse', 'acute', 'right', or 'invalid'.
    """
    angles = (a, b, c)
    if (sum(angles) == 180 and
        all([0 < a < 180 for a in angles])):
            if any([a > 90 for a in angles]):
                return "obtuse"
            if all([a < 90 for a in angles]):
                return "acute"
            if 90 in angles:
                return "right"
    return "invalid"
```
Obtuse ![](images/obtuse_triangle.png),
Acute ![](images/acute_triangle.png),
Right ![](images/right_triangle.png)
???
* Here's a function `triangle_type` that takes 3 angles and returns 4 possible values.
* If you know the difference between obtuse and acute triangles, you rock.
* I had to look it up.
---

# without Parametrization

`test_1_without_param.py`:

```python
from triangle import triangle_type

def test_obtuse():
    assert triangle_type(100, 40, 40) == "obtuse"

def test_acute():
    assert triangle_type(60, 60, 60) == "acute"

def test_right():
    assert triangle_type(90, 60, 30) == "right"

def test_invalid():
    assert triangle_type(0, 0, 0) == "invalid"
```
Obtuse ![](images/obtuse_triangle.png),
Acute ![](images/acute_triangle.png),
Right ![](images/right_triangle.png)
???
* Since there are 4 possible outcomes, we need at least 4 test cases.
* There's a lot of redundancy there.
* If our tests were more complicated, this gets even more painful.
* Adding new test cases often involves cut/paste/modify, which can produce errors.
---
# Function Parametrization
`test_2_func_param.py`:
```python
import pytest
from triangle import triangle_type

many_triangles = [
    (100, 40, 40, "obtuse"),
    ( 60, 60, 60, "acute"),
    ( 90, 60, 30, "right"),
    (  0,  0,  0, "invalid"),
]

@pytest.mark.parametrize('a, b, c, expected', many_triangles)
def test_type(a, b, c, expected):
    assert triangle_type(a, b, c) == expected
```
???
* using parametrize, we can test the same 4 cases with one test function.
* the list of parameter rows can be inline, I've split it out to make the syntax easier to see.
---

# output without

```
(venv) $ pytest -v test_1_without_param.py 
===================== test session starts ======================
collected 4 items                                              

test_1_without_param.py::test_obtuse PASSED              [ 25%]
test_1_without_param.py::test_acute PASSED               [ 50%]
test_1_without_param.py::test_right PASSED               [ 75%]
test_1_without_param.py::test_invalid PASSED             [100%]

====================== 4 passed in 0.02s =======================
```
---

# output with

```bash
(venv) $ pytest -v test_2_func_param.py 
===================== test session starts ======================
collected 4 items                                              

test_2_func_param.py::test_type[100-40-40-obtuse] PASSED [ 25%]
test_2_func_param.py::test_type[60-60-60-acute] PASSED   [ 50%]
test_2_func_param.py::test_type[90-60-30-right] PASSED   [ 75%]
test_2_func_param.py::test_type[0-0-0-invalid] PASSED    [100%]

====================== 4 passed in 0.02s =======================
```
---

# You can still run just one

```bash
(venv) $ pytest -v 'test_2_func_param.py::test_type[0-0-0-invalid]'
===================== test session starts ======================
collected 1 item                                               

test_2_func_param.py::test_type[0-0-0-invalid] PASSED    [100%]

====================== 1 passed in 0.02s =======================
```
This uses the node id.

---

# or more

```bash
(venv) $ pytest -v -k 60 test_2_func_param.py
===================== test session starts ======================
collected 4 items / 2 deselected / 2 selected                  

test_2_func_param.py::test_type[60-60-60-acute] PASSED   [ 50%]
test_2_func_param.py::test_type[90-60-30-right] PASSED   [100%]

=============== 2 passed, 2 deselected in 0.02s ================
```
An example with `-k` to pick all tests with 60 degree angles.
---
# Once again, this is Function Parametrization
`test_2_func_param.py`:
```python
import pytest
from triangle import triangle_type

many_triangles = [
    (100, 40, 40, "obtuse"),
    ( 60, 60, 60, "acute"),
    ( 90, 60, 30, "right"),
    (  0,  0,  0, "invalid"),
]

*@pytest.mark.parametrize('a, b, c, expected', many_triangles)
*def test_type(a, b, c, expected):
    assert triangle_type(a, b, c) == expected
```
---


# Fixture Parametrization

`test_3_fixture_param.py`:
```python
import pytest
from triangle import triangle_type

many_triangles = [
    (100, 40, 40, "obtuse"),
    ( 60, 60, 60, "acute"),
    ( 90, 60, 30, "right"),
    (  0,  0,  0, "invalid"),
]

*@pytest.fixture(params=many_triangles)
*def a_triangle(request):
*    return request.param

*def test_type(a_triangle):
*    a, b, c, expected = a_triangle
    assert triangle_type(a, b, c) == expected
```
???
* syntax is a little different
* a fixture is referenced by the test, a_triangle
* the a_triangle fixture is parametrized with the same list we used before
* instead of the values a, b, c, expected being referenced in the param list, 
  each row is passed in as the value of request.param
* in this case, I'm passing it along to the test with return request.param
* the test then is unpacking the tuple into the separate values.
* the assert line is the same, though.
---
# output


```bash
(venv) $ pytest -v test_3_fixture_param.py 
======================= test session starts =======================
collected 4 items                                                 

test_3_fixture_param.py::test_type[a_triangle0] PASSED      [ 25%]
test_3_fixture_param.py::test_type[a_triangle1] PASSED      [ 50%]
test_3_fixture_param.py::test_type[a_triangle2] PASSED      [ 75%]
test_3_fixture_param.py::test_type[a_triangle3] PASSED      [100%]

======================== 4 passed in 0.02s ========================
```
???
* Since the parametrization is an object, a tuple in this case, pytest
doesn't try to guess how to reference it, and just assigns the cases
increasing numbers. 
* Not really helpful. 
* This happens with function parametrization as well if you use objects in the parameter list.
* ids solve this
---
# adding an id function

`test_4_fixture_param.py`:
```python
import pytest
from triangle import triangle_type

many_triangles = [
    (100, 40, 40, "obtuse"),
    ( 60, 60, 60, "acute"),
    ( 90, 60, 30, "right"),
    (  0,  0,  0, "invalid"),
]

*def idfn(a_triangle):
*    a, b, c, expected = a_triangle
*    return f'{a}_{b}_{c}_{expected}'

*@pytest.fixture(params=many_triangles, ids=idfn)
def a_triangle(request):
    return request.param

def test_type(a_triangle):
    a, b, c, expected = a_triangle
    assert triangle_type(a, b, c) == expected
```
???
* here I show a custom `idfn()`.
* however, often `repr` or `str` are fine id functions.
* and would have worked here just fine  

---
# output

```bash
(venv) $ pytest -v test_4_fixture_param.py 
======================= test session starts =======================
collected 4 items                                                 

test_4_fixture_param.py::test_type[100_40_40_obtuse] PASSED [ 25%]
test_4_fixture_param.py::test_type[60_60_60_acute] PASSED   [ 50%]
test_4_fixture_param.py::test_type[90_60_30_right] PASSED   [ 75%]
test_4_fixture_param.py::test_type[0_0_0_invalid] PASSED    [100%]

======================== 4 passed in 0.02s ========================
```

---

# `pytest_generate_tests()`
`test_5_gen.py`:
```python
from triangle import triangle_type

many_triangles = [
    (100, 40, 40, "obtuse"),
    ( 60, 60, 60, "acute"),
    ( 90, 60, 30, "right"),
    (  0,  0,  0, "invalid"),
]

def idfn(a_triangle):
    a, b, c, expected = a_triangle
    return f'{a}_{b}_{c}_{expected}'

*def pytest_generate_tests(metafunc):
*    if "a_triangle" in metafunc.fixturenames:
*        metafunc.parametrize("a_triangle",
*                             many_triangles,
*                             ids=idfn)

def test_type(a_triangle):
    a, b, c, expected = a_triangle
    assert triangle_type(a, b, c) == expected
```
???
* the third method is pytest_generate_tests
* very similar to fixture parametrization
* here I'm using the same fixed list, many_triangles
* howerver, this hook has access to other things like command line arguments, so 
  you do have an opportunity to build up the parametrization list
  during runtime. 
---
# output
```bash
(venv) $ pytest -v test_5_gen.py 
=================== test session starts ====================
platform darwin -- Python 3.7.3, pytest-5.2.1, py-1.8.0, pluggy-0.13.0 -- /Users/okken/projects/presentation_parametrization/venv/bin/python
cachedir: .pytest_cache
rootdir: /Users/okken/projects/presentation_parametrization/code
collected 4 items                                          

test_5_gen.py::test_type[100_40_40_obtuse] PASSED    [ 25%]
test_5_gen.py::test_type[60_60_60_acute] PASSED      [ 50%]
test_5_gen.py::test_type[90_60_30_right] PASSED      [ 75%]
test_5_gen.py::test_type[0_0_0_invalid] PASSED       [100%]

==================== 4 passed in 0.03s =====================
```
???
Output should look familiar.
---
# Choosing a Technique

Guidelines

1. **function parametrization**
    * use this if you can
2. **fixture parametrization**
    * if doing work to set up each fixture value
    * if cycling through pre-conditions
    * if running multiple test against the same set of "setup states"
3. **pytest_generate_tests()**
    * if you need to build up the list at runtime 
    * if list is based on passed in parameters or external resources
    * for sparse matrix sets
???
1. **function parametrization**
    * use this if you can
2. **fixture parametrization**
    * if doing work to set up each fixture value
    * if cycling through pre-conditions
    * if running multiple test against the same set of "setup states"
3. **pytest_generate_tests()**
    * if you need to build up the list at runtime 
    * if list is based on passed in parameters or external resources
    * for sparse matrix sets
    
---

# more test cases

`test_6_more.py`:
```python
...
triangles = [
    (  1, 1, 178, "obtuse"), # big angles
    ( 91, 44, 45, "obtuse"), # just over 90
    (0.01, 0.01, 179.98, "obtuse"), # decimals 

    (90, 60, 30, "right"), # check 90 for each angle
    (10, 90, 80, "right"),
    (85,  5, 90, "right"),

    (89, 89,  2, "acute"), # just under 90
    (60, 60, 60, "acute"),

    (0, 0, 0, "invalid"),     # zeros
    (61, 60, 60, "invalid"),  # sum > 180
    (90, 91, -1, "invalid"),  # negative numbers
]

@pytest.mark.parametrize('a, b, c, expected', triangles)
def test_type(a, b, c, expected):
    assert triangle_type(a, b, c) == expected
``` 
???
* parametrization 
    * easy to have a more full set of test cases 
    * without much extra work 
    * without much increased maintenance costs
* in this instance, 4 test cases really isn't enough
* this is a more realistic set of test cases

---
# Combining Techniques

You can have multiple parametrizations for a test function.

* can have multiple `@pytest.mark.parametrize()` decorators on a test function.
* can parameterize multipe fixtures per test
* can use pytest_generate_tests() to parametrize multiple parameters
* can use a combination of techniques 
* can blow up into lots and lots of test cases very fast
???
You can have multiple parametrizations for a test function.

* can have multiple `@pytest.mark.parametrize()` decorators on a test function.
* can parameterize multipe fixtures per test
* can use pytest_generate_tests() to parametrize multiple parameters
* can use a combination of techniques 
* can blow up into lots and lots of test cases very fast
---
# Not covered, intentionally

Use with caution.

* indirect 
    * Have a list of parameters defined at the test function that gets passed to a fixture. 
    * Kind of a hybrid between function and fixture parametrization.
* subtests 
    * Sort of related, but not really.
    * You can check multiple things within a test.
    * If you care about test case counts, pass, fail, etc, then don't use subtests.
???
Use with caution.

* indirect 
    * Have a list of parameters defined at the test function that gets passed to a fixture. 
    * Kind of a hybrid between function and fixture parametrization.
* subtests 
    * Sort of related, but not really.
    * You can check multiple things within a test.
    * If you care about test case counts, pass, fail, etc, then don't use subtests.

---

# Resources

.left-two-thirds[
* [Python Testing with pytest](https://t.co/AKfVKcdDoy?amp=1) 
    * The fastest way to get super productive with pytest
* pytest docs on 
    * [parametrization, in general](https://docs.pytest.org/en/latest/parametrize.html) 
    * [function parametrization](https://docs.pytest.org/en/latest/parametrize.html#pytest-mark-parametrize)
    * [fixture parametrization](https://docs.pytest.org/en/latest/fixture.html#fixture-parametrize)
    * [pytest_generate_tests](https://docs.pytest.org/en/latest/parametrize.html#basic-pytest-generate-tests-example)
* podcasts
    * [Test & Code](https://testandcode.com) 
    * [Python Bytes](https://pythonbytes.fm)
    * [Talk Python](https://talkpython.fm)
* slack community: [Test & Code Slack](https://testandcode.com/slack) 
* Twitter: [@brianokken](https://twitter.com/brianokken), [@testandcode](https://twitter.com/testandcode)
* This code, and markdown for slides, on [github](https://github.com/okken/presentation_parametrization)
* Presentation will go on slideshare soon
]
.right-third[
<a href="https://t.co/AKfVKcdDoy?amp=1">
<img src="images/book.jpg" style="border-style: solid;" width="200">
</a>
]