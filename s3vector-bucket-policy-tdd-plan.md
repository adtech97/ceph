# S3Vector bucket policy TDD plan

## Overview

This document outlines a test-driven path to add vector bucket policy support and policy enforcement for existing S3Vector APIs. The scope covers the missing vector bucket policy REST APIs and permission checks for [`RGWS3VectorGetVectors::verify_permission()`](src/rgw/rgw_rest_s3vector.cc:473), [`RGWS3VectorPutVectors::verify_permission()`](src/rgw/rgw_rest_s3vector.cc:422), and [`RGWS3VectorDeleteVectors::verify_permission()`](src/rgw/rgw_rest_s3vector.cc:946). The approach is to begin with characterization tests in [`src/test/rgw/s3vectors/s3vector_test.py`](src/test/rgw/s3vectors/s3vector_test.py), use those results to confirm the current request and policy flow, then implement the missing REST and backend pieces with the smallest possible changes before tightening the tests into positive and negative policy scenarios.

## Sub-task 1

- **Intent** — Add a first end-to-end characterization test matrix that exercises cross-user access to vector bucket resources and records the current behavior before policy support is implemented.
- **Expected Outcomes** — [`src/test/rgw/s3vectors/s3vector_test.py`](src/test/rgw/s3vectors/s3vector_test.py) contains baseline tests for `get_vectors`, `put_vectors`, and `delete_vectors` using an owner user and a second user. The tests are structured so they can later be tightened into allow and deny assertions once policy support is added.
- **Todo List**
  1. Add shared helper setup in [`src/test/rgw/s3vectors/s3vector_test.py`](src/test/rgw/s3vectors/s3vector_test.py) to create a vector bucket, create an index, and seed vectors for reuse across policy-flow tests.
  2. Add one baseline cross-user test each for get, put, and delete vectors that captures either success or `ClientError` and records the returned status and error code.
  3. Keep the initial assertions broad enough to characterize the current flow without assuming final policy semantics.
  4. Document in test comments that these tests will later be converted into deny-by-default and allow-by-policy cases.
- **Relevant Context** — Existing test helpers live in [`connection()`](src/test/rgw/s3vectors/s3vector_test.py:67), [`another_user()`](src/test/rgw/s3vectors/s3vector_test.py:110), and the CRUD/index tests around [`test_create_vector_bucket()`](src/test/rgw/s3vectors/s3vector_test.py:148) and [`test_create_index()`](src/test/rgw/s3vectors/s3vector_test.py:403).
- **Status** — [ ] pending

### Example test helper shape

```python
def _setup_vector_bucket_with_index(conn, bucket_name, index_name, dimension=4):
    result = conn.create_vector_bucket(vectorBucketName=bucket_name)
    assert result['ResponseMetadata']['HTTPStatusCode'] == 200

    result = conn.create_index(
        vectorBucketName=bucket_name,
        indexName=index_name,
        dataType='float32',
        dimension=dimension,
        distanceMetric='euclidean')
    assert result['ResponseMetadata']['HTTPStatusCode'] == 200

    vectors = [
        {'key': 'v0', 'data': [0.0, 1.0, 2.0, 3.0]},
        {'key': 'v1', 'data': [4.0, 5.0, 6.0, 7.0]},
    ]
    result = conn.put_vectors(
        vectorBucketName=bucket_name,
        indexName=index_name,
        vectors=vectors)
    assert result['ResponseMetadata']['HTTPStatusCode'] == 200
```

### Example baseline test shape

```python
@pytest.mark.vector_test
def test_vector_bucket_policy_flow_get_vectors_baseline():
    owner = connection()
    other = another_user()
    bucket_name = gen_bucket_name()
    index_name = 'test-index'

    _setup_vector_bucket_with_index(owner, bucket_name, index_name)

    try:
        result = other.get_vectors(
            vectorBucketName=bucket_name,
            indexName=index_name,
            keys=['v0'])
        status = result['ResponseMetadata']['HTTPStatusCode']
        error_code = None
    except other.exceptions.ClientError as err:
        status = err.response['ResponseMetadata']['HTTPStatusCode']
        error_code = err.response['Error']['Code']

    assert status in (200, 403, 404)
```

## Sub-task 2

- **Intent** — Collate evidence from the baseline tests and map it back to the current request path so implementation is guided by observed behavior instead of assumptions.
- **Expected Outcomes** — A short implementation note is added to this document after the baseline run that records which of the three vector APIs currently succeed or fail cross-user, and whether the behavior is consistent with the missing permission checks and missing vector bucket policy support.
- **Todo List**
  1. Run the new baseline tests using the workflow documented in [`src/test/rgw/s3vectors/README.rst`](src/test/rgw/s3vectors/README.rst:5).
  2. Record the observed status code and error code for each baseline test in this markdown file under a new results section before code changes begin.
  3. Compare those observations against the current S3Vector request path in [`RGWHandler_REST_s3Vector::read_permissions()`](src/rgw/rgw_rest_s3vector.h:22), which bypasses normal permission state loading.
  4. Use the observations to decide whether the first implementation step should focus on policy API storage, permission-state preparation, or both.
- **Relevant Context** — Permission-state loading for normal bucket flows happens through [`RGWHandler_REST::init_permissions()`](src/rgw/rgw_rest.cc:1859), [`RGWHandler_REST::read_permissions()`](src/rgw/rgw_rest.cc:1869), and [`rgw_build_bucket_policies()`](src/rgw/rgw_op.cc:566). S3Vector currently overrides [`read_permissions()`](src/rgw/rgw_rest_s3vector.h:22) with a no-op.
- **Status** — [ ] pending

## Sub-task 3

- **Intent** — Implement the missing vector bucket policy APIs so tests can drive resource policy storage and retrieval before enforcement is added to data APIs.
- **Expected Outcomes** — The following stubs are implemented end to end: [`RGWS3VectorPutVectorBucketPolicy`](src/rgw/rgw_rest_s3vector.cc:865), [`RGWS3VectorGetVectorBucketPolicy`](src/rgw/rgw_rest_s3vector.cc:903), [`RGWS3VectorDeleteVectorBucketPolicy`](src/rgw/rgw_rest_s3vector.cc:379), plus their backend helpers in [`src/rgw/rgw_s3vector.cc`](src/rgw/rgw_s3vector.cc). Tests can put, get, and delete a policy document on a vector bucket.
- **Todo List**
  1. Inspect the standard bucket policy implementation pattern in [`RGWPutBucketPolicy::execute()`](src/rgw/rgw_op.cc:9400), [`RGWGetBucketPolicy::execute()`](src/rgw/rgw_op.cc:9474), and [`RGWDeleteBucketPolicy::execute()`](src/rgw/rgw_op.cc:9531).
  2. Implement the vector bucket policy backend helpers in [`src/rgw/rgw_s3vector.cc`](src/rgw/rgw_s3vector.cc) using the existing vector bucket metadata path and the bucket policy attribute key [`RGW_ATTR_IAM_POLICY`](src/rgw/rgw_common.h:177).
  3. In the three S3Vector REST ops, finish `init_processing()` where needed so `vector_bucket_arn` is synthesized when absent and `s->bucket_name` is set when useful for downstream policy logic.
  4. Implement `execute()` bodies and response bodies/status codes for put, get, and delete policy using the same external API shape as the documented AWS calls.
  5. Add focused tests in [`src/test/rgw/s3vectors/s3vector_test.py`](src/test/rgw/s3vectors/s3vector_test.py) for put, get, and delete vector bucket policy round trips.
- **Relevant Context** — Policy request structures already exist in [`put_vector_bucket_policy_t`](src/rgw/rgw_s3vector.h:400), [`get_vector_bucket_policy_t`](src/rgw/rgw_s3vector.h:415), and [`delete_vector_bucket_policy_t`](src/rgw/rgw_s3vector.h:137). Their backend stubs are in [`src/rgw/rgw_s3vector.cc`](src/rgw/rgw_s3vector.cc:1031) and nearby decode helpers.
- **Status** — [ ] pending

### Example implementation sketch for policy storage

```cpp
int put_vector_bucket_policy(const put_vector_bucket_policy_t& configuration,
                             DoutPrefixProvider* dpp,
                             optional_yield y) {
  // load vector bucket metadata/attrs
  // parse policy text using rgw::IAM::Policy
  // store policy in attrs[RGW_ATTR_IAM_POLICY]
  // persist attrs back to the vector bucket
}
```

## Sub-task 4

- **Intent** — Wire policy enforcement into existing vector data APIs with the minimal shared setup needed for `req_state` to carry the policy context used by [`verify_bucket_permission()`](src/rgw/rgw_common.h:1856).
- **Expected Outcomes** — [`RGWS3VectorGetVectors::verify_permission()`](src/rgw/rgw_rest_s3vector.cc:473), [`RGWS3VectorPutVectors::verify_permission()`](src/rgw/rgw_rest_s3vector.cc:422), and [`RGWS3VectorDeleteVectors::verify_permission()`](src/rgw/rgw_rest_s3vector.cc:946) enforce the corresponding `rgw::IAM::s3vectors...` actions against the vector bucket policy and identity policies.
- **Todo List**
  1. Confirm whether vector bucket metadata can populate the same `req_state` fields used by [`verify_bucket_permission()`](src/rgw/rgw_common.cc:1500), especially `s->bucket_attrs`, `s->bucket_acl`, `s->bucket_owner`, and `s->iam_policy`.
  2. If needed, add a small vector-specific preparation path before `verify_permission()` so those fields are loaded from the vector bucket rather than a regular bucket.
  3. Update `init_processing()` in get, put, and delete vectors so `vector_bucket_arn` is always available before authorization.
  4. Replace the commented TODO checks with explicit calls to [`verify_bucket_permission(this, s, configuration.vector_bucket_arn.get(), action)`](src/rgw/rgw_common.h:1856).
  5. Keep the change minimal and limit it to the three APIs in scope unless the baseline or shared setup proves another API must be touched.
- **Relevant Context** — S3Vector currently bypasses normal permission reads in [`RGWHandler_REST_s3Vector::read_permissions()`](src/rgw/rgw_rest_s3vector.h:22). Existing partial bucket-style checks are in [`RGWS3VectorDeleteVectorBucket::verify_permission()`](src/rgw/rgw_rest_s3vector.cc:319) and [`RGWS3VectorGetVectorBucket::verify_permission()`](src/rgw/rgw_rest_s3vector.cc:671).
- **Status** — [ ] pending

### Example enforcement sketch

```cpp
int verify_permission(optional_yield y) override {
  if (!verify_bucket_permission(this, s,
        configuration.vector_bucket_arn.get(),
        rgw::IAM::s3vectorsGetVectors)) {
    return -EACCES;
  }
  return 0;
}
```

## Sub-task 5

- **Intent** — Convert the baseline tests into final TDD assertions that validate the complete bucket policy process across policy CRUD and data API authorization.
- **Expected Outcomes** — The test suite covers the full sequence: policy creation, policy retrieval, policy deletion, deny-by-default or no-policy behavior, allow behavior for an authorized second user, and deny behavior after policy removal or explicit deny.
- **Todo List**
  1. Tighten the three baseline cross-user tests into explicit expectations once the observed behavior and implementation are stable.
  2. Add positive-policy cases where user A stores a vector bucket policy that allows user B to call `get_vectors`, `put_vectors`, and `delete_vectors`.
  3. Add negative-policy cases where the same calls fail without policy or after policy deletion.
  4. If the final implementation supports explicit deny semantics, add one explicit deny case for at least one of the three APIs.
  5. Run the relevant s3vectors tests as documented in [`src/test/rgw/s3vectors/README.rst`](src/test/rgw/s3vectors/README.rst:5) and record any notable setup requirements in this markdown file.
- **Relevant Context** — The s3vectors test runner and invocation examples are documented in [`src/test/rgw/s3vectors/README.rst`](src/test/rgw/s3vectors/README.rst:5). The suite lives in [`src/test/rgw/s3vectors/s3vector_test.py`](src/test/rgw/s3vectors/s3vector_test.py).
- **Status** — [ ] pending
