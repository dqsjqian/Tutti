# Changelog

## 2.1.1

- Wait for accepted bulk tasks before rethrowing a later submission failure,
  keeping their borrowed callbacks alive until every chunk finishes.
- Support sized random-access ranges that expose iterators without `operator[]`.
- Avoid signed distance, rounded chunk-size and offset overflow in `parallel_for`,
  including ranges that span the complete signed index domain.
- Document that `wait()` observes an idle pool and can include concurrent work.
- Verify header version constants against the CMake project version.
