# To Mock or Not to Mock - Mocking Unit Tests

**Author**: [Lee Napthine](/team/lee-napthine)  
**Last updated:** 2025-04-24

**Originally published at:** [https://arcsoft.uvic.ca/log/2025-03-14-mocking-unit-tests/](https://arcsoft.uvic.ca/log/2025-03-14-mocking-unit-tests/)

If you’ve ever written unit tests, you’ve probably encountered situations where a
function depends on an external API, a database, or a cloud service. These things are difficult
to control in a test environment. This is where mocking comes in.

Mocks speed up testing, cover for third-party dependencies, and let us focus on what matters: our
code. With Python tools like `Mock()` and `MagicMock()`, we have flexibilty with how we write our tests.

In this log, we’ll dive into:

- When to use mocks and when not to
- The difference between `MagicMock()`, `Mock()`, and `mocker.patch()`
- Real examples from our CALM project

Let’s start by exploring when to use mock tests.

## When to Use Mock Tests

My first real introduction to `Mock()` came when I was asked to review and expand a
set of tests that modified cloud resources allocated in an OpenStack environment. Running tests
that could permanently alter private resources was a non-starter. The solution? Dive into
Python’s confusing, but ultimately powerful, set of mock tools for testing third-party
dependencies.

Mock tests come into play whenever we need to test code that interacts with external
dependencies, such as:

- Cloud services
- Databases
- APIs

If your code depends on external system you don’t control, mocking lets you simulate responses
and run tests in a controlled environment. You can sprinkle mocks into your standard unit tests
or dedicate entire test suites to them.

Okay. So, it’s obvious that mocks are useful. But what’s less obvious is how over-mocking
can deliver misleading test results and false confidence.

## The Dangers of Mock Tests

To use mocks we often contruct fake data structures, classes, methods, and even objects. Since
mocks behave exactly as we tell them to, they will always return the expected results. This can
easily lead us to have false confidence in the stability of the code base.

If you run an entire test suite of mocks, like we did for our Openstack program, you might be
skirting around the real logic. Here me must be careful we don’t run into false positives
where our code is broken. One of the stronger ways to prevent this from happening is to use [functional testing](https://medium.com/@cpinjala/understanding-false-positives-in-test-implementation-a-practical-guide-e8e8a6e9accb) which verifies outputs based on their inputs.

Lastly, if we depend on third-party apps, syncing can become troublesome. For instance, if you
configured your mock tests to pass while mocking certain versions of a dependency, if that
dependency changes and your code fails, your tests will continue to mimick the prior error-free
results. This means test maintenance is necessary for evolving dependencies.

The best use of mock test will be to use them sparingly (if possible) and thread them [throughout
real integrated testing](https://www.hypertest.co/integration-testing/mocking-and-stubbing-in-api-integration-testing#:~:text=Why%20Mock%20and%20Stub%20in%20API%20Integration%20Testing%3F).

## What Mock Tool to Use

Python’s `unittest.mock` provides several types of mocks, each with its own use case.
The most common ones are `Mock()`, `MagicMock()`, and `mocker.patch()`. But what are their differences? While we won’t cover all the functionality these test frameworks provide in one log, we do need to get an understanding of when to use patching, side effects, and tracking calls.

According to the [Python documentation](https://docs.python.org/dev/library/unittest.mock-examples.html), common use cases include:

- Replacing methods to check if they are called with the correct arguments.
- Recording method calls on objects to verify behavior.
- Mocking class or object instantiations to control return values.
- Tracking and asserting.

### Mock() vs. MagicMock()

Both `Mock()` and `MagicMock()` allow us to replace dependencies and verify their usage with all the important `assert` and `assert_called_once_with()` methods. The big difference is `MagicMock()` extends `Mock()` with built-in support for magic methods (e.g., `__len__`, `__getitem__`, `__iter__`).

In most of these examples the Mock and MagicMock classes are interchangeable. As the [MagicMock
is the more capable class](https://docs.python.org/dev/library/unittest.mock-examples.html#mock-for-method-calls-on-an-object) it makes a sensible default choice, but we found it useful to use both. Read on for why.

#### Why We Use MagicMock() in calm Tests

In the CALM test suite (the name of our Openstack cloud program), we use `MagicMock()` when dealing with objects that:

- Have dynamic attributes (e.g., `.compute.servers.return_value`)
- Might use `MagicMocks()` magic methods
- Are complex objects like OpenStack resources, which need a more flexible mock

Example from a `mock_conn()` fixture in our project:

```python
@pytest.fixture
def mock_conn():
    """Mock OpenStack connection."""
    mock_conn = MagicMock()
    mock_conn.compute.servers.return_value = []  # this simulates an Openstack API call
    return mock_conn
```

- The above example allows us to mock OpenStack’s `compute.servers` without
  manually defining compute as a sub-mock.
- If we used `Mock()`, we’d need to explicitly create `mock_conn.compute = Mock()` before setting `servers.return_value`.
- `MagicMock()` auto-creates attributes dynamically as needed.

```python
@pytest.fixture
def mock_cloud(mock_conn):
    """Mock Cloud instance."""
    with patch("openstack.connect", return_value=mock_conn):
        return Cloud(config=MagicMock(cloud="mock"))
```

- Here, `MagicMock(cloud="mock")` ensures the config object behaves like
  a real config object.
- If it needed special attributes or functions (e.g., `__getitem__`), `MagicMock()` would support them automatically.

#### Why We Use Mock() in calm Tests

In these examples, we use `Mock()` when we need explicit control over what is being
mocked and don’t require magic methods.

Example of mocking an object’s behavior in our `test_scan_command()`:

```python
def test_scan_command(mock_cloud, mock_conf, mock_args, mocker):
    """
    Test a basic scan works and produces an expected report.
    """
    mock_info = mocker.patch("calm.scanner.info")  # use `Mock()` for specific function calls
    result = calm.scanner.scan_cmd(mock_conf, mock_args)

    assert result == calm.cli.OK
    mock_info.assert_any_call("Scanned 2 projects, skipping 1 which are exempt.")
    mock_info.assert_any_call(
        "Scanned 4 instances in eligible projects, finding 3 instances eligible for CALM."
    )
```

- `mock_info = mocker.patch("calm.scanner.info")` allows us to check
  exactly what arguments `calm.scanner.info` is called with.
- We don’t need magic methods here—just an assertion on method calls.
- Using `MagicMock()` wouldn’t add any benefit, since `info()` isn’t a
  complex object with attributes or special methods.

## How to Use Mocking Techniques in Testing

### Mocking objects

In unit testing, mock objects simulate dependencies that are hard to test. Instead of relying on
real objects that might have unpredictable behavior, we create a stand-in.

For our CALM project, we needed to mock an OpenStack cloud environment, including
projects, instances, and metadata, without actually connecting to OpenStack. This required the
mock to emulate real API responses:

```python
fake_projects = dotdictify({
    'project1': {
        'name': "project1",
        'id': "PROJECT1",
        'tags': [calm.TAG_EXEMPT],
        'instances': [
            {
                'name': "instance1-1", "id": "ID-111",
                'flavor': {'original_name': 'p2-48gb'},
                'tags': [], 'metadata': {},
                'created_at': NOW, 'updated_at': NOW
            }
        ]
    }
})
```

OpenStack’s API responses allow attribute-style access (`instance.name`) rather than
dictionary-style (`instance[name]`). Using `dotdictify()`, we mimic this
behavior in our tests without modifying the code base. You can see how this demonstrates
realistic test cases while ensuring our mocks behave like actual OpenStack data structures.

### Call Tracking

Lastly, let’s go over [call tracking](https://docs.python.org/3/library/unittest.mock-examples.html#tracking-all-calls) and how it was used in our project example:

```python
def test_mark_exempt(mock_cloud, mock_conn):
    """Test marking a server as exempt."""
    mock_server = MagicMock(spec=openstack.compute.v2.server.Server)
    mock_project = MagicMock(spec=openstack.identity.v3.project.Project)

    mock_cloud.mark_exempt(mock_server)
    mock_cloud.mark_exempt(mock_project)

    mock_server.add_tag.assert_called_once_with(mock_conn.compute, calm.TAG_EXEMPT)
    mock_project.add_tag.assert_called_once_with(mock_conn.identity, calm.TAG_EXEMPT)

    expected_calls = [
        call.mark_exempt(mock_server),
        call.mark_exempt(mock_project)
    ]
    assert mock_cloud.mock_calls == expected_calls
```

`mock_calls` keeps a record of all calls made to the mock object. In our case, `mock_calls` is determining that all these assertions and calls to the Openstack API have occured the desired amount of times and in the correct order. `assert_called_once_with` confirms that the call happens a single time, and `assert` confirms that the call retrieved the expected result.

## Summary

To sum up:

1. Create fake data structures.
2. Create fake methods.
3. Emulate results and verify behavior.

With the interconnectness of modern applications, mocking is increasingly relevant. In our case,
an entire test suite was mocked, but when possible, balance mocks with real unit tests.

Your tests may pass, but that doesn’t mean your real API calls will work. Functional
testing matches inputs to their expected outputs and is the recommended approach to using mock
tests. Remember, with mocking you have ultimate control. This is both the strength and the
danger of using them.

You have several choices of libraries to use in Python to mock tests. Use `Mock()` if
you need more control, and use `MagicMock()` if you want the convenience of having
built-in support for magic methods.

I hope this was helpful! Good luck testing.
