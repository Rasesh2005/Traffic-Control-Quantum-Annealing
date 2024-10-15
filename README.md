# Traffic Management

Let’s look into the problem statement and assumptions I made:

- We know the current location and destination of every vehicle in the road graph.
- No vehicle will intentionally stop in between; i.e., no traffic is caused by accidents and other mishaps. (In such cases, we may temporarily delete that road for the time being.)
- Emergency vehicles have the highest priority; i.e., even if 100 vehicles are waiting on another side of a traffic signal, the lane with an ambulance will be allowed to pass first.
- All the traffic lights in the road graph are being controlled by a single entity; i.e., their values depend on each other.

## INPUT DATA

- **Road Network Data:** A detailed graph of the city’s road network including lanes, speed limits of every road, intersections, and traffic signal data.
- **Vehicle Data:** Real-time information on every vehicle's positions, speeds (assumed to be at speed limit in my case), and volumes (number of cars at each traffic light intersection) from sensors or GPS systems.
- **Event Information:** Locations of events and the expected peak times of increased traffic (may be to simply start the navigation and traffic light system).
- **Emergency Vehicle Data:** Real-time positions of critical vehicles that require priority routes.

## OUTPUT DATA

- Optimized traffic signal timings at intersections.
- Suggested rerouting paths for vehicles to avoid congested areas.
- Prioritized, congestion-free routes for emergency vehicles (just a small change to accommodate emergency vehicles into the system).

## METHOD USED

**Quadratic Unconstrained Binary Optimization (QUBO)**

D-Wave Systems has quantum annealers that we can use to minimize Binary Quadratic Equations (basically 2-degree equations in which each variable is binary, i.e., it can be either 0 or 1). The D-Wave Solver requires us to create a Binary Quadratic Model from the equation we formed and pass it, giving the value for each variable at which the minimum value of the model (BQM) is attained.

Also, for binary variables, we know \(x^2 = x\).

Formally, the QUBO model is expressed by the optimization problem:

$$
\text{QUBO: minimize/maximize } y = x^T Q x
$$

where \(x\) is a vector of binary decision variables and \(Q\) is a square matrix of constants.

## APPLICATION OF METHOD

### Solving Navigation Problem Using QUBO

Expressed as pseudo-code, the important high-level steps of the traffic flow optimization algorithm are as follows:

1. For each car \(i\):
   - Determine the current route.
2. For each car \(i\)’s current route:
   - Map the source and destination to their nearest nodes in the road graph.
3. For each source/destination pair:
   - Determine all simple paths from source to destination.
   - Find two alternative paths that are maximally dissimilar to the original route and to each other.
4. For each car \(i\), define the set of possible routes needed to form the QUBO.
5. Define the matrix \(Q\) with binary variables \(q_{ij}\) as described below.
6. Solve the QUBO problem.
7. Update cars with the selected routes.

Now coming to the QUBO matrix:

We define a route as the list of streets it needs to go through to the final destination. We take 5 alternative routes and calculate for each one the best route for each vehicle.

For every possible assignment of car to route, we define a binary variable \(q_{ij}\) representing car \(i\) taking route \(j\). Because each car can only occupy one route at a time, exactly one variable per car must be true in the minimum of the QUBO.

Note: We cannot have no car taking a route, obviously. So we exclude that case.

So we use the constraint term (or penalty term):

$$
0 = \left( \sum_{j \in \{1, 2, 3\}} q_{ij} - 1 \right)^2 = -q_{i1} - q_{i2} - q_{i3} + 2q_{i1}q_{i2} + 2q_{i2}q_{i3} + 2q_{i1}q_{i3} + 1
$$

The cost of a single street can be calculated as follows:

$$
\text{cost}(s) = \left( \sum_{q_{ij} \in B_s} q_{ij} \right)^2.
$$

So adding the penalty for all cars and multiplying by the total coefficient, and also adding up the cost of all roads in the traffic network, we receive the final objective function which we need to minimize:

$$
\text{Obj} = \sum_{s \in S} \text{cost}(s) + \lambda \sum_{i} \left( \sum_{j} q_{ij} - 1 \right)^2.
$$

### Solving Traffic Light

Let’s first define the binary variables for solving the QUBO problem.

Let \(x_{ij}\) be the \(i\)th intersection in mode \(j\); i.e., if the \(i\)th intersection is in the \(j\)th mode, then \(x_{ij}=1\), else \(0\).
<img width="760" alt="image1" src="https://github.com/user-attachments/assets/de84dac1-4eae-48df-a957-8f092a73f021">
Now we multiply constants \(C_{ij}\) with each \(x_{ij}\), indicating weightage, i.e., the number of vehicles that will exit the intersection if \(x_{ij}=1\).

Now, with this, we get the expression for each intersection as follows:

$$
\text{Obj} = -\sum_{i=1}^{n} \sum_{j=1}^{6} C_{ij} x_{ij}^2
$$

Now we need to find if we can find a relation between different intersections. Let the average time taken by any vehicle from one intersection \(i\) to another intersection \(i'\) be \(\Delta t_{ii'}\). It has to go in a specific mode (from the navigation path we decoded above). Then if

$$
t \mod \Delta t_{ii'} \approx 0
$$

we can say that these two intersections can be interrelated to each other; that is, only for adjacent crossings.

Let \(\tau_{ii'}\) be 1 if the above condition is satisfied, else zero. We get the equation:

$$
\lambda_2 \sum_{i=1}^{n} \sum_{j=1}^{6} C_{ij} x_{ij} \left[ \tau_{i,a'} \lambda_3 C_{a'a} x_{a',a} + \tau_{i,b'} \lambda_3' C_{b'b} x_{b',b} + \tau_{i,c'} \lambda_3 C_{c'c} x_{c',c} + \tau_{i,d'} \lambda_3' C_{d'd} x_{d',d} \right]
$$

For the penalty term, we need for a crossing \(i\) only one of the 6 modes \(j\) to be active, so we add the penalty term as follows:

$$
\lambda_4 \sum_{i} \left[ 1 - \sum_{j=1}^{6} x_{ij} \right]^2
$$

Now the final equation is as follows:

$$
\begin{aligned}
\text{Obj} &= -\lambda_1 \sum_{i=1}^{n} \sum_{j=1}^{6} C_{ij} x_{ij}^2 \\
& \quad - \lambda_2 \sum_{i=1}^{n} \sum_{j=1}^{6} C_{ij} x_{ij} \left[ \tau_{i,a'} \lambda_3 C_{a'a} x_{a', a} + \tau_{i,b'} \lambda_3' C_{b'b} x_{b',b} + \tau_{i,c'} \lambda_3 C_{c'c} x_{c', c} + \tau_{i,d'} \lambda_3' C_{d'd} x_{d',d} \right] \\
& \quad + \lambda_4 \sum_{i} \left[ 1 - \sum_{j=1}^{6} x_{ij} \right]^2
\end{aligned}
$$

This is the objective function we need to minimize (we put a minus in front of all terms we need to maximize and a plus in front of terms we need to minimize). \(\lambda_3\) is the weightage given to each neighboring intersection from the current intersection.

In the case of emergency vehicles, we multiply \(q_{ij}\) (the \(i\)th vehicle being the emergency one) with a very large weight (let's say 102) so that the objective function gets a higher penalty for longer routes or wait times at traffic signals.

## PIPELINE

1. Extract all the road data and get the alternate routes for every vehicle (maximum 4-5 alternate routes with similar ETA at no traffic).
2. Formulate QUBO for traffic lights and navigation, calculate the best possible route for each vehicle to follow, and pass it to the GPS of every vehicle.
3. Formulate QUBO for traffic lights and pass the signal with timing to the traffic light control system with a timestamp.
4. Note that before passing the signal from each qubo solver, we need to decode what the output of qubo solver gave(from the definitions we set in the Application section).  
5. In Case of emergency vehicles, either the city contains bus exclusive lane, and they may use that for zero traffic, or in less urban areas, we can give high weightage to the emergency vehicles(let's say 100, a number which is greater than maximum number of cars that can be in a single road) to clear its path asap.
