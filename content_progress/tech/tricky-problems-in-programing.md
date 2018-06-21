There are few crazy and tricky problems in computer science that are hard to deal with. I want to discuss few of them here.
These problems are tricky because they brings ambiguity, uncertainty and confusion due to common perception. When they cause an error in code, it is hard to detect them and requires significant efforts to deal with.

1. Date Format Detection d/m/y vs m/d/y
The valid date time format detection in Random Dataset is one of the difficult problem in data parsing. Let's consider an example of a date 2/2/2015. It could be either in d/m/y format or m/d/y format. Given only one value point, there is no way to tell which one is correct. Generally we need to look through list of dates and find the one that parsed by only either d/m/y or m/d/y to declare that format as winner. i.e. if you find entry that looks like 23/12/2015 in list of date points, you can be sure that list is of d/m/y format. In highly unlike case, if you have no tie breaker value in list, there is no way but to ask data provider to know correct date format. How to stay away from trouble? Always insist on generating and using dates in ISO format that also works on different time zone.

2. Floating point precision
The floating point precision is also one of the oldest and complex problem in computer science. Thera has been few standards defined around representing floating point in 32bit or 64bit format i.e. IEEE 756. However, things get tricky when floating points are used with arithmetic operations because of their internal representation in computer hardware. Always be careful when things go wrong and datapoint of type floats are involved. Specifically during the float values comparison operation because those few off bits (due to level of precision or different internal representation) gives False Negative results.
More discussion on python's official documentation.
https://docs.python.org/3.6/tutorial/floatingpoint.html
