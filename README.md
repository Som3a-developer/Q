#include <iostream>
#include <omp.h>

using namespace std;

int main()
{
    int sum = 0;
    int count = 0;

    #pragma omp parallel for reduction(+:sum,count)
    for(int i = 1; i <= 1000; i++)
    {
        if(i % 2 != 0)
        {
            sum += i;
            count++;
        }
    }

    double avg = (double)sum / count;

    cout << "Average = " << avg << endl;

    return 0;
}
#include <iostream>
#include <fstream>
#include <vector>
#include <string>
#include <cstdlib>
#include <omp.h>

using namespace std;

int main()
{
    ifstream input("operations.txt");

    vector<string> operations;
    string op;

    
    while(getline(input, op))
    {
        operations.push_back(op);
    }

    input.close();

    ofstream output("results.txt");

    #pragma omp parallel for
    for(int i = 0; i < operations.size(); i++)
    {
        int a = rand() % 100 + 1;
        int b = rand() % 100 + 1;

        int result = 0;

        if(operations[i] == "+")
            result = a + b;

        else if(operations[i] == "-")
            result = a - b;

        else if(operations[i] == "*")
            result = a * b;

        else if(operations[i] == "/")
            result = a / b;

        
        #pragma omp critical
        {
            output << a << " "
                   << operations[i] << " "
                   << b << " = "
                   << result << endl;
        }
    }

    output.close();

    cout << "Done" << endl;

    return 0;
}
