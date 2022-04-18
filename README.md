# Movebase with Sparse-Waypoint Goals

This package has been derived from the original move_base node from
[ROS Navigation package](http://wiki.ros.org/navigation). It implements
the `move_base_swp` node, which extends the interface of the the original
`move_base` node to include the sparse waypoints that the robot must
visit along its route to the goal.

'move_base_swp' is a drop-in replacement for `move_base`,
designed to work with all existing plugins (e.g. DWA planner, navfn,
costmap2d) from the Navigation package. The interface is fully compatible
with `move_base` and will accept single-pose goals and behave
just as regular `move_base` with a couple of improvements listed below.
The new interface accepts the sparse-waypoint goals which specifies a
list of poses, with last pose being the final goal. If a single-waypoint
goal is provided the goal reduces to a final goal with no waypoints only
issued over the new interface. All other components will work, so
`move_base_swp` is simply a better `move_base`.

## Feature Overview

In this section, we describe the improvements that `move_base_swp`
implements on the top of original `move_base` node.

### Sparse-waypoint goals

`move_base_swp` implements a new `actionlib` interface, which allows
the caller to define the goal in form of a list of waypoints. Unlike dense
waypoints generated by the global planner, this list is sparse. Typical size
of this list will include no more than 5-10 waypoints, sometimes just one.
The path between sparse waypoints is still within the responsibility
of the global planner. In other words, the input to `move_base_swp`
is a list of sparse waypoints that represent an the goal along with
a list of interim checkpoints that the robot must visit along its path.
The global planner produces a list of dense waypoints in the same
manner as in the original `move_base`, which are fed to the local planner,
which, in turn, produces the velocity vector that creates the motion.

While it is theoretically possible to supply a dense set of waypoints
over the interface and effectively override the global planner, this is
not intended usage. Such usage will likely result in poor performance
because of frequent calls to the global planner. The intended usage is
for situations when the robot must get from point A to point B, but the
application requires that it visits point C along the way. Simply issuing
the goal at point B would result in finding the shortest path within the
constraints of the cost map which may or may not include point C. If the
point C is on a "detour" path it will not be visited. Using `move_base_swp`
allows the application to specify this additional constraint.

While it is possible to achieve the described behavior by first issuing
the goal to point C and then as the robot approaches that goal issue
the new goal to point B, this will likely result in unwanted slowdown
or even complete stop at the interim waypoint. Further, an application
would have to constantly monitor the progress of the first goal and
issue the next goal "just in time" to ensure smooth transition. Often
the "just in time" would imply using ill-defined heuristics that heavily
depend on local and global planner tuning parameters.

`move_base_swp` solves the problem by first calling the global planner
for each segment of the path (A-to-C segment, followed by C-to-B segment
in our example) and then concatenating the two plans into one global-planner
solution before feeding it to the local planner. The result is a smooth
motion at the concatenation point without the need for application
to intervene in any way. The application simply follows the progress
towards the specified goal which includes the specified sparse waypoints.

### Windowed global plan

`move_base_swp` provides an option to feed the horizon of the global
plan that is being fed into the local planner. Instead of feeding
an entire global plan, `move_base_swp` has an option to feed only
a specified number of plan waypoints starting from the present location
and load additional plan waypoints as the robot progresses towards the goal.

This feature can improve motion behavior in general and can stand on
its own regardless of whether sparse-waypoint plan is used or not. However,
the problem that the windowed global plan solves is exacerbated by the
usage of sparse waypoints.

An example of the problem that exists even in the original `move_base`
package is illustrated in this video:

https://youtu.be/7vnxZ25MzEk

The robot is moving from one side to the other of an X-shaped obstacle.
Notice how at 0:57 it enters the state where it is not making progress.
This happens because the entire area of interest is within the local
planner window (we used DWA planner for this experiment). The local planner
has the choice to either produce the velocity vector that takes it along
the route prescribed by the global plan (the top path shown in green)
or to deviate from the global plan and move the robot down the bottom
path. From the local planner's perspective, both paths have equal cost
for the set of parameters it is using and the solver is bouncing back
and forth between two options without making much progress.

While one motion may eventually prevail, the time it will take for this
to happen, is not bounded. Changing the weight factors in the DWA planner
to prefer sticking to the global plan more than advancing towards the
target (i.e. cutting corners) would resolve this case, but there is always
another case for which the same phenomena would happen. Triggering the
patience timeout on the local planner and requesting the replan will
in practice often break this live-lock, but it would be preferable if
we could avoid the need to timeout. Making the local planner horizon
smaller would also help in this case, but at the expense of degraded
obstacle avoidance performance.

This video shows how the windowed global plan solves the problem:

https://youtu.be/VPTIrmf1TQQ

The thin green line shows the plan as determined by the global
planner, which goes all the way to the goal. The thick green line shows
the section of the global plan to which the local planner has been exposed.
As the robot makes progress, the `move_base_swp` feeds additional waypoints
to the local planner. The new waypoints can be fed either continuously
or when the number of presently used waypoints runs below the low-watermark
threshold. Because the local planner is only exposed to limited number
of waypoints, effectively cutting its horizon, the secondary viable
path that causes the live-lock has been eliminated.

The next video shows how the usage of sparse waypoints can be at odds
with the local planner:

https://youtu.be/x_Khtj9rBhc

The robot is in the open space (no obstacles) and it gets a goal with
two waypoints that define a zig-zag path to the final pose. If the local
planner sees the whole plan, it will eventually start cutting corners
as the opportunistic behavior of reducing the distance to the goal starts
to prevail. The effect is especially visible for the second waypoint
where the robot does not even attempt to visit it, but instead cuts
straight to the goal.

This video shows how the windowed global plan solves the problem:

https://youtu.be/8t-57WI_6UY

By exposing the local planner to a shorter subset of the global plan,
the robot is forced to head towards the waypoint, before it learns about
the existence of the sharp turn and gives the opportunity to the local
planner to cut the corner.

### Brake rampdown

If the goal is canceled or preempted, or the robot is made stop by any
means other than arriving to the goal, the original `move_base` node would
abruptly bring the velocity vector to zero. This can result in a jerky
motion, which can cause the stress or even damage to robot's mechanical
or electrical system. `move_base_swp` introduces the break-rampdown feature
which makes the robot slow down gradually. The rampdown rate is controlled
by a parameter and the behavior can be turned off, essentially resulting
on original `move_base` behavior.

### Handbrake interface

Sometimes, an application may decide to temporarily stop the robot
and then after some time allow it to proceed towards its target. In the
original `move_base` the only way to achieve this is to cancel
the goal and re-issue the new goal after the application decides to
restore the motion. `move_base_swp` provides a dedicated handbrake
interface that implements this feature. When the robot is ordered to
handbrake, it will not cancel the goal. The goal and the planner will
remain active until the handbrake is released, after which the robot
will continue to pursue its original plan.

### Code improvements

Besides extended functionality, `move_base_swp` also refactors and
improves the general `move_base` code. Many instances of repeated code
are extracted into common functions. Blocks of code embedded in
long functions are extracted into separate functions and named accordingly.
Finally, some locks are reworked to improve the interlocking correctness.

## Interfaces

`move_base_swp` provides all interfaces that `move_base` does unchanged.
If the `move_base` node in an application designed to work with it is
replaced with the `move_base_swp` node, the behavior will remain identical.
Other than launching the new mode no change to the application would
be necessary. However, to use the new capabilities (described above), the
application would have to be extended to use the new interfaces. This
makes the transition to `move_base_swp` easy, because the node can be
first swapped out, followed by gradually converting the application to
use the new features.

This section describes the interfaces (topics and `actionlib` interfaces)
that allow the application to use the new capabilities of the `move_base_swp'

### Sparse-waypoint `actionlib` interface.

This is a standard `actionlib` interface. Actions are defined in
'move_base_swp/action/MoveBaseSWP.action` file. The standard topics associated
with this interface are as follows, `move_base_swp/goal`,
`move_base_swp/result`, `move_base_swp/status`, `move_base_swp/feedback`, and
`move_base_swp/cancel`. The goal supplied is the list of poses that
represent the waypoints that the robot should visit. The last pose
in the list is the final goal:

```
geometry_msgs/PoseStamped[] waypoint_poses
```

If a single-element list is provided as the goal, the resulting behavior
will be that of the original `move_base`. The
[legacy interface](http://wiki.ros.org/move_base) is also available and
consists of original topics `move_base/goal`, `move_base/result`,
`move_base/status`, `move_base/feedback`, and `move_base/cancel`. The
goal received on the legacy `move_base` interface will be type-converted
and forwarded to the `move_base_swp` interface. The feedback and the result
will be visible on both interface.

The goal received on the new `move_base_swp`
interface will be executed natively and the feedback and result will
be visible on the new interface only. Issuing the new goal on either
of the two interfaces will preempt the goal in progress regardless
of the interface the goal was issued on. Only one goal can be pursued
at the time, regardless of the interface.

The simple-goal topic (`/move_base_simple/goal`) is also available for
sending goals from RVIZ.

### Navigation status topics

In addition to the `actionlib` interface, `move_base_swp` also
provides several topics that can be used to monitor and visualize the
progress of the goal. All topics that exist in the
[original `move_base`](http://wiki.ros.org/move_base)
node exist in the `move_base_swp` and they are not repeated here.

The namespace used for all topics is `move_base` (not `move_base_swp`)
because these topics are meant to be used together with original
topics provided by the `move_base`.

The new topics are:

`/move_base/current_waypoints` ([`geometry_msgs/PoseArray`](http://docs.ros.org/en/api/geometry_msgs/html/msg/PoseArray.html))
>This topic contains the list of waypoints that the robot is currently
pursuing. As the robot visits each waypoint, it is removed from the list.
The order of poses in the list is the order in which the waypoints will be
visited.

`/move_base/snapped_pose` ([`geometry_msgs/PoseStamped`](http://docs.ros.org/en/api/geometry_msgs/html/msg/PoseStamped.html))
>This topic represents the robot pose (as derived from the transform tree)
snapped to the global plan that the robot is pursuing. The snapping
is done by finding the nearest point in 2D space that lies on the line
composed out of dense waypoints generated by the global planner. This pose
can be used to visualize the robot progress along the plan. Internally
`move_base_swp` uses this pose to determine when to feed new segment
of the global plan into the local planner
(see [Windowed global plan](#windowed-global-plan) section for details).

`/move_base/pursued_plan` ([`nav_msgs/Path`](http://docs.ros.org/en/api/nav_msgs/html/msg/Path.html))
> This topic is the subset of the global plan that has been fed to
the local planner and that the robot is currently pursuing. As the
robot makes progress along this plan, the waypoints are replenished
from the global plan (published by the global planner).

### Handbrake topic

`/move_base/handbrake` ([`std_msgs/Bool`](http://docs.ros.org/en/api/std_msgs/html/msg/Bool.html))
> Sending `True` to this topic will pull the handbrake stopping the robot
along its plan. Sending `False` will release the handbrake. The handbrake
must be renewed every second (or less) by re-sending `True` to the topic.
This is required to prevent applications from halting the robot and
disappearing. Note that this is not a safety brake (a safety brake would
require an inverse logic that stops the robot if not renewed). This
handbrake is meant to be used as part of the robot navigation strategy
(e.g. stopping to yield to another robot).
