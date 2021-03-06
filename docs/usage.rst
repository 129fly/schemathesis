.. _usage:

Usage
=====

Storing network requests
------------------------

For debugging purposes, it can be useful to store all requests generated by Schemathesis and all responses from the app into
a separate file. Schemathesis allows you to do this with ``-store-network-log`` command-line option:

.. code:: bash

    $ schemathesis run --store-network-log=cassette.yaml http://127.0.0.1/schema.yaml

This command will create a new YAML file that will network interactions in `VCR format <https://relishapp.com/vcr/vcr/v/5-1-0/docs/cassettes/cassette-format>`_.
It might look like this:

.. code:: yaml

    command: 'schemathesis run --store-network-log=cassette.yaml http://127.0.0.1/schema.yaml'
    recorded_with: 'Schemathesis 1.2.0'
    http_interactions:
    - id: '0'
      status: 'FAILURE'
      seed: '1'
      elapsed: '0.00123'
      recorded_at: '2020-04-22T17:52:51.275318'
      request:
        uri: 'http://127.0.0.1/api/failure'
        method: 'GET'
        headers:
          ...
        body:
          encoding: 'utf-8'
          base64_string: ''
      response:
        status:
          code: '500'
          message: 'Internal Server Error'
        headers:
          ...
        body:
          encoding: 'utf-8'
          base64_string: 'NTAwOiBJbnRlcm5hbCBTZXJ2ZXIgRXJyb3I='
        http_version: '1.1'

Schemathesis provides the following extra fields:

- ``command``. Full CLI command used to run Schemathesis.
- ``http_interactions.id``. A numeric interaction ID within the current cassette.
- ``http_interactions.status``. Type of test outcome, is one of ``SUCCESS``, ``FAILURE``, ``ERROR``.
- ``http_interactions.seed``. Hypothesis seed used in that particular case could be used as an argument to ``--hypothesis-seed`` CLI option to reproduce this request.
- ``http_interactions.elapsed``. Time in seconds that a request took.

To work with the cassette you could use `yq <https://github.com/mikefarah/yq>`_ or any similar tool.
Show response body content of first failed interaction:

.. code:: bash

    $ yq r foo.yaml 'http_interactions.(status==FAILURE).response.body.base64_string' | head -n 1 | base64 -d
    500: Internal Server Error

Check payload in requests to ``/api/upload_file``:

.. code:: bash

    $ yq r foo.yaml 'http_interactions.(request.uri==http://127.0.0.1/api/upload_file).request.body.base64_string' | base64 -d
    --7d4db38ad065994d913cb02b2982e3ba
    Content-Disposition: form-data; name="data"; filename="data"


    --7d4db38ad065994d913cb02b2982e3ba--

Replaying interactions
----------------------

Saved cassettes can be replayed with ``schemathesis replay`` command. Additionally, you may filter what interactions to
replay by these parameters:

- ``id``. Specific, unique ID;
- ``status``. Replay only interactions with this status (``SUCCESS``, ``FAILURE`` or ``ERROR``);
- ``uri``. A regular expression for request URI;
- ``method``. A regular expression for request method;

During replaying Schemathesis will output interactions being replayed together with the response codes from the initial and
current execution:

.. code:: bash

    $ schemathesis replay foo.yaml --status=FAILURE
    Replaying cassette: foo.yaml
    Total interactions: 4005

      ID              : 0
      URI             : http://127.0.0.1:8081/api/failure
      Old status code : 500
      New status code : 500

      ID              : 1
      URI             : http://127.0.0.1:8081/api/failure
      Old status code : 500
      New status code : 500

Export CLI test results to JUnit.xml
------------------------------------

It is possible to export test results to format, acceptable by such tools as Jenkins.

.. code:: bash

    $ schemathesis run --junit-xml=/path/junit.xml http://127.0.0.1/schema.yaml

This command will create an XML at given path as in example below.

.. code:: xml

    <?xml version="1.0" ?>
    <testsuites disabled="0" errors="1" failures="1" tests="3" time="0.10743043999536894">
        <testsuite disabled="0" errors="1" failures="1" name="schemathesis" skipped="0" tests="3" time="0.10743043999536894" hostname="bespin">
            <testcase name="GET /api/failure" time="0.089057">
                <failure type="failure" message="2. Received a response with 5xx status code: 500"/>
            </testcase>
            <testcase name="GET /api/malformed_json" time="0.011977">
                <error type="error" message="json.decoder.JSONDecodeError: Expecting property name enclosed in double quotes: line 1 column 2 (char 1)
    ">Traceback (most recent call last):
      File &quot;/home/user/work/schemathesis/src/schemathesis/runner/impl/core.py&quot;, line 87, in run_test
        test(checks, targets, result, **kwargs)
      File &quot;/home/user/work/schemathesis/src/schemathesis/runner/impl/core.py&quot;, line 150, in network_test
        case: Case,
      File &quot;/home/user/.pyenv/versions/3.8.0/envs/schemathesis/lib/python3.8/site-packages/hypothesis/core.py&quot;, line 1095, in wrapped_test
        raise the_error_hypothesis_found
      File &quot;/home/user/work/schemathesis/src/schemathesis/runner/impl/core.py&quot;, line 165, in network_test
        run_checks(case, checks, result, response)
      File &quot;/home/user/work/schemathesis/src/schemathesis/runner/impl/core.py&quot;, line 133, in run_checks
        check(response, case)
      File &quot;/home/user/work/schemathesis/src/schemathesis/checks.py&quot;, line 87, in response_schema_conformance
        data = response.json()
      File &quot;/home/user/.pyenv/versions/3.8.0/envs/schemathesis/lib/python3.8/site-packages/requests/models.py&quot;, line 889, in json
        return complexjson.loads(
      File &quot;/home/user/.pyenv/versions/3.8.0/lib/python3.8/json/__init__.py&quot;, line 357, in loads
        return _default_decoder.decode(s)
      File &quot;/home/user/.pyenv/versions/3.8.0/lib/python3.8/json/decoder.py&quot;, line 337, in decode
        obj, end = self.raw_decode(s, idx=_w(s, 0).end())
      File &quot;/home/user/.pyenv/versions/3.8.0/lib/python3.8/json/decoder.py&quot;, line 353, in raw_decode
        obj, end = self.scan_once(s, idx)
    json.decoder.JSONDecodeError: Expecting property name enclosed in double quotes: line 1 column 2 (char 1)
    </error>
            </testcase>
            <testcase name="GET /api/success" time="0.006397"/>
        </testsuite>
    </testsuites>


Base URL configuration
----------------------

If your Open API schema defines ``servers`` (or ``basePath`` in Open API 2.0), then these values will be used to
construct full endpoint URL during testing. In case of Open API 3.0 the first value from ``servers`` will be used.

However, you may want to run tests against a different base URL. To do this you need to pass ``--base-url`` option in CLI
or provide ``base_url`` argument to a loader / runner if you use Schemathesis in your code:

.. code:: bash

    schemathesis run --base-url=http://127.0.0.1:8080/api/v2 http://production.com/api/openapi.json

And if your schema defines ``servers`` like this

.. code:: yaml

    servers:
      - url: https://production.com/api/{basePath}
        variables:
          basePath:
            default: v1

Then the tests will be executed against ``/api/v2`` base path.
