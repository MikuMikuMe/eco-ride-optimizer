# Eco-Ride-Optimizer

Creating an intelligent system for Eco-Ride-Optimizer involves several key components, including data handling, route optimization, and error handling. A common approach for such a project would involve using libraries for route optimization and APIs for actual map data. Below is a simplified Python program framework using popular libraries such as Google Maps API for routing and `ortools` for route optimization. Note that you need a Google Maps API key to make requests.

```python
import googlemaps
from ortools.constraint_solver import routing_enums_pb2
from ortools.constraint_solver import pywrapcp
from itertools import combinations

# Initialize your Google Maps API client
gmaps = googlemaps.Client(key='YOUR_GOOGLE_MAPS_API_KEY')

class RideSharingOptimizer:
    def __init__(self, locations):
        # List of addresses or coordinates (latitude, longitude)
        self.locations = locations
        self.distance_matrix = None

    def calculate_distances(self):
        """ Calculate distance matrix using Google Maps Distance Matrix API. """
        try:
            # Call to the Distance Matrix API
            distance_matrix_result = gmaps.distance_matrix(origins=self.locations,
                                                           destinations=self.locations,
                                                           mode="driving")

            # Extract the distance information and populate the matrix
            self.distance_matrix = [
                [
                    element['distance']['value'] for element in row['elements']
                ] for row in distance_matrix_result['rows']
            ]
        except Exception as e:
            print(f"An error occurred while calculating distances: {e}")
    
    def create_data_model(self):
        """ Create the data model for the routing optimizer. """
        data = {}
        data['distance_matrix'] = self.distance_matrix
        data['num_vehicles'] = 1
        data['depot'] = 0
        return data

    def solve(self):
        """ Solve the routing problem using Google's OR-Tools. """
        try:
            data = self.create_data_model()
            manager = pywrapcp.RoutingIndexManager(len(data['distance_matrix']),
                                                   data['num_vehicles'], data['depot'])
            routing = pywrapcp.RoutingModel(manager)

            # Create and register a transit callback
            def distance_callback(from_index, to_index):
                from_node = manager.IndexToNode(from_index)
                to_node = manager.IndexToNode(to_index)
                return data['distance_matrix'][from_node][to_node]

            transit_callback_index = routing.RegisterTransitCallback(distance_callback)
            routing.SetArcCostEvaluatorOfAllVehicles(transit_callback_index)

            # Set search parameters
            search_parameters = pywrapcp.DefaultRoutingSearchParameters()
            search_parameters.first_solution_strategy = (
                routing_enums_pb2.FirstSolutionStrategy.PATH_CHEAPEST_ARC)

            # Solve the problem
            solution = routing.SolveWithParameters(search_parameters)

            if solution:
                # Fetch and print the route
                index = routing.Start(0)
                plan_output = 'Route:\n'
                route_distance = 0
                while not routing.IsEnd(index):
                    plan_output += f' {manager.IndexToNode(index)} ->'
                    previous_index = index
                    index = solution.Value(routing.NextVar(index))
                    route_distance += routing.GetArcCostForVehicle(previous_index, index, 0)
                plan_output += ' 0\n'
                plan_output += f'Distance of the route: {route_distance} meters\n'
                print(plan_output)
            else:
                print('No solution found!')

        except Exception as e:
            print(f"An error occurred during the optimization process: {e}")

if __name__ == "__main__":
    # List of locations to visit (addresses or LatLng tuples)
    locations = [
        "123 Main Street, Anytown, USA", 
        "456 Maple Avenue, Anytown, USA", 
        "789 Oak Street, Anytown, USA"
    ]

    optimizer = RideSharingOptimizer(locations)
    optimizer.calculate_distances()

    if optimizer.distance_matrix:
        optimizer.solve()
    else:
        print("Failed to compute the distance matrix.")
```

### Key Points:
- **Google Maps API:** Make sure to replace `'YOUR_GOOGLE_MAPS_API_KEY'` with an actual API key.
- **Error Handling:** The program includes basic error handling to catch and report issues during API calls and optimization.
- **Simplified Scope:** Given this is a high-level project, the current program is illustrative and focuses on the core idea.
- **Data Model & Solver:** Set up a data model compatible with OR-Tools and implement constraints, such as using multiple vehicles or adding constraints like maximum ride time, etc.

These steps provide a foundational framework upon which more sophisticated features (like dynamic ride pairing) can be developed.