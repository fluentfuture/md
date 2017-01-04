This page introduces a few advanced ways of using Mockito spies.

## Invariants and State-based Objects

* Do implement object invariants with spies.
* Do not mock and `when().thenReturn()` invariants.

Mock object can be a great tool if used properly. But when the test double has invariants to be respected, mocking isn't almost effective at this kind of job.

As demonstrated in [Mocks are Stupid](http://endoflineblog.com/testing-with-doubles-or-why-mocks-are-stupid-part-4), HttpServletRequest is one such object with invariants:

```java
HttpServletRequest reqStub = mock(HttpServletRequest.class);
when(reqStub.getParameterMap()).thenReturn(ImmutableMap.of(key, new String[]{val}));
```
Mocking or stubbing getParameterMap() causes brittle tests because there are several different ways in which the subject under test's implementation could interact with the request object.
It could call `request.getParameterMap().get(key)`, or `request.getParameterValues(key)`.

Similarly, the SUT could call `setAttribute(key, value)` and then later the code would expect to read it through `getAttribute(key)`; or expect to get a null if `removeAttribute(key)` was called on some code path. None of that is mock object's strong suit.

Typically, test doubles with invariants are better created as fakes: proper Java classes that override methods with behavior, except, well, fakes are not cheap.

HttpServletRequest has what? 30 or 40 methods? And who's to say it won't gain more?

Ideally, for common interfaces like HttpServletRequest, there should already be a FakeHttpServletRequest classes created and tested by someone so we can just use for free. But I can say for sure I've been caught off guard on large interfaces that no one has bothered to create a proper fake implementation and I surely wasn't up for such task when all I needed was to test my SUT that uses only 2 or 3 methods.

Mockito's `@Spy` offers a middle-ground where you don't have to give up one benefit for the other. Say, I know my SUT only deals with attributes, such fake implementation is rather trivial to implement, with @Spy, because I can just override the 3 methods I care about:

```java
@Spy private FakeHttpServletRequest request;

@Test public void testBadAttributeCauseAutoLogout() {
  request.addAttribute(MAGIC_ATTRIBUTE_KEY, "bad");
  new LoginServlet().service(request, response);
  verify(request).logout();
}

static abstract class FakeHttpServletRequest implements HttpServletRequest {
  private finl Map<String, Object> attributes = new LinkedHashMap<>();

  @Override public Object getAttribute(String name) {
    return attributes.get(name);
  }

  @Override public Enumeration<String> getAttributeNames() {
    return Iterators.asEnumeration(attributes.keySet().iterator());
  }

  @Override public void setAttribute(String name, Object value) {
    attributes.put(name, value);
  }
}
```java
And did you see that line calling `verify(request).logout()` on the spy? It means that not only can we implement plain old Java methods for better invariant handling, we don't lose out the ability to use it as a mock where mocks work better: testing interactions (in this case, logout() be called once and only once).


Another example where @Spy helps to create better test code: suppose we have a Job scheduler framework that internally utilizes a `ScheduledExecutorService` to invoke the jobs at the right time.

To test such framework, manually programing the ScheduledExecutorService.schedule() and submit() methods and all the time-related logic would be tedious, at best.

The following test uses `@Spy` to test that a job failed with SERVER_TOO_BUSY gets rescheduled for a later run:
```java
public class JobSchdulerTest {
  @Spy private FakeClock clock;
  @Spy private FakeScheduledExecutorService executor;
  @Mock private Task task;

  @Test public void testJobsFailedWithTooBusyGetsRescheduled() {
    scheduler().add(task, Duration.ofMillis(10));
    elapse(Duration.ofMillis(9));
    // should not have run
    verify(task, never()).run();

    // at scheduled time, it runs, but failed with TOO_BUSY
    when(task.run()).thenReturn(SERVER_TOO_BUSY);
    elapse(Duration.ofMillis(1));
    verify(task).run(); 
    reset(task);

    // so it's retried.
    when(task.run()).thenReturn(OK);
    elapse(Duration.ofMillis(20));
    verify(task).run();
    reset(task);

    // Retry succeeded. No more runs.
    elapse(Duration.ofMillis(10000));
    verify(task, never()).run();
  }

  private JobScheduler scheduler() {
    return new JobScheduler(clock, executor);
  }

  // Moves time and invokes ready jobs.
  private void elapse(Duration time) {
    clock.elapse(time);
    executor.run();
  }

  static abstract class FakeClock extends Clock {
    private Instant now = Instant.ofEpochMilli(0);

    @Override public Instant instant() {
      return now;
    }

    void elapse(Duration duration) {
      now = now.plus(duration);
    }
  }

  abstract class FakeScheduledExecutorService
      implements ScheduledExecutorService {
    private List<Job> jobs = new ArrayList<>();

    @Override public ScheduledFuture<?> schedule(
        Runnable command, long delay, TimeUnit unit) {
      jobs.add(new Job(command, delay, unit));
    }

    /** Runs all jobs that are ready by now. Leaves the rest. */
    void run() {
      Instant now = clock.instant();
      List<Job> ready =
          jobs.stream().filter(job -> job.ready(now)).collect(toList());
      jobs = jobs.stream()
          .filter(job -> job.pending(now))
          .collect(toCollection(ArrayList::new));
      ready.forEach(Job::run);
    }
  }
}
```
While it requires a bit of code in the FakeScheduledExecutorService class, we get to extract orthogonal logic out of the tests so that tests stay clean and easy to understand. And in reality, the fake will be reused by many different tests in the test class.

It's worth noting that the class being spied is allowed to be non-static innner class of the test class, which enables it to read state from other fields, in this case, the clock object. Using this technique, we make the two objects working together seamlessly.

## Dummy objects

* Do use spy to create dummies
* Do not return a mock from helper methods.

A somewhat common mistake is as reported in this Stack Overflow [thread](http://stackoverflow.com/questions/26318569/unfinished-stubbing-detected-in-mockito). That is, trying to use a factory helper that returns a mock while configuring another mock. The code can look quite innocent and puzzling:
```java
Model model = mock(Model.class);
when(model.getSubModel()).thenReturn(dummySubModel());

private SubModel dummySubModel() {
  SubModel sub = mock(SubModel.class);
  when(sub.getName()).thenReturn("anything but null");
  return sub;
}
```

What happens essentially if dummySubModel() were inlined looks like this:
```java
Model model = mock(Model.class);
when(model.getSubModel());
SubModel sub = mock(SubModel.class);
when(sub.getName());  // Oops!
```
the second when() call happened before the first when() call is finished.

For helpers that create dummy objects like this, using `spy()` minimizes caveats and surprises down the road when other people inevitably attempt to use your helper method, because there is no `when()` call involved:
```java
Model model = mock(Model.class);
when(model.getSubModel()).thenReturn(dummySubModel());

private SubModel dummySubModel() {
  return spy(DummySubModel.class);
}

static abstract class DummySubModel implements SubModel {
  @Override public String getName() {
    return "anything but null";
  }
}
```
