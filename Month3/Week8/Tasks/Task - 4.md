
## Choice of debugger

GDB, on C++. This is my primary language for competitive programming and the debugger I already use when a bug survives reading the code. Compiled with `-O0 -g` so that line numbers and locals are accurate.

## The bug

Codeforces 2014E ("Robin Meets Marian"): shortest path where `h` vertices have horses that halve travel time for the rest of the journey. Standard approach is Dijkstra over an augmented state space of `(vertex, hasHorse)`, run once from vertex 1 and once from vertex n, then take `min over v of max(d1[v], dn[v])`.

Symptom: **TLE, not WA.** The code produced correct answers on every sample and on my own tests. It was just too slow.

That detail is what makes this a debugger problem rather than a print-statement problem. There was no wrong value to print. Nothing in the output was suspicious. The only observable defect was elapsed time, so adding `cerr` lines would have shown me a stream of entirely correct intermediate state.

## The TLE code

Breakpoints are marked as comments (`// BP-1`, `// BP-2`).

```cpp
#include <bits/stdc++.h>
using namespace std;
typedef long long ll;
#define all(x) (x).begin(), (x).end()
#define allr(x) (x).rbegin(), (x).rend()
#define endl '\n'
#define LOOKWHOSBACK                  \
    ios_base::sync_with_stdio(false); \
    cin.tie(NULL);                    \
    cout.tie(NULL);
#define pb push_back
#define Yes cout << "Yes"
#define No cout << "No"
#define YES cout << "YES"
#define NO cout << "NO"
#define nl cout << "\n";
#define print(x)      \
    for (auto &y : x) \
        cout << y << ' ';
#define debug(...) cerr << "(" << #__VA_ARGS__ << ") = ", dbg(__VA_ARGS__)
void dbg() { cerr << endl; }
template <typename T, typename... Args>
void dbg(T first, Args... rest)
{
    cerr << first << " ";
    dbg(rest...);
}
#define debugArr(arr)      \
    cerr << #arr << " = "; \
    for (auto x : arr)     \
        cerr << x << " ";  \
    cerr << endl;
#define trace() cerr << __FUNCTION__ << " called at line " << __LINE__ << endl;
#define NDEBUG // Disable assertions

#include <ext/pb_ds/assoc_container.hpp>
#include <ext/pb_ds/tree_policy.hpp>
using namespace __gnu_pbds;
typedef tree<int, null_type, less_equal<int>, rb_tree_tag,
             tree_order_statistics_node_update> ordered_multiset;

const ll oo = 1e18;
const int mod = 998244353;

void solve() {
    // BP-1
    int n, m, h;
    cin >> n >> m >> h;
    vector<bool> horses(n, false);
    for(int i = 0; i < h; ++i) {
        int x;
        cin >> x;
        x--;
        horses[x] = true;
    }
    vector<vector<pair<int, int> > > adj(n + 1);
    for(int i = 0; i < m; ++i) {
        int u, v, w;
        cin >> u >> v >> w;
        u--, v--;
        adj[u].push_back({v, w});
        adj[v].push_back({u, w});
    }

    auto dijkstra = [&](int src) {
        vector<array<ll, 2> > dist(n, {oo, oo});
        priority_queue<tuple<ll, int, int>, vector<tuple<ll, int, int> >,
                       greater<tuple<ll, int, int> > > pq;
        dist[src][0] = 0;
        pq.push({0, src, 0});

        while(!pq.empty()) {
            auto[cst, node, hasHorse] = pq.top();
            pq.pop();
            if(cst > dist[node][hasHorse]) continue;
            if(horses[node] && !hasHorse && cst < dist[node][1]) {
                dist[node][1] = cst;
                pq.push({cst, node, 1});
            }
            for(auto &[to, w]: adj[node]) {
                for (auto &[to, w] : adj[node]) {   // <-- the bug
                ll nw = hasHorse ? w / 2 : w;
                    if (cst + nw < dist[to][hasHorse]) {
                        dist[to][hasHorse] = cst + nw;
                        pq.push({cst + nw, to, hasHorse});
                    }
                }
            }
        }
        return dist;
    };

    auto d1 = dijkstra(0);
    auto dn = dijkstra(n - 1);

    if (min(d1[n - 1][0], d1[n - 1][1]) >= oo) {
        cout << -1;
        return;
    }
    // BP-2
    ll ans = oo;
    for (int i = 0; i < n; ++i) {
        ll marian = min(d1[i][0], d1[i][1]);
        ll robin  = min(dn[i][0], dn[i][1]);
        if (marian >= oo || robin >= oo) continue;
        ans = min(ans, max(marian, robin));
    }
    cout << ans;
}

int main()
{
    LOOKWHOSBACK
    int t = 1;
    cin >> t;
    while (t--)
    {
        solve();
        cout << "\n";
    }
    return 0;
}
```

## The session

**BP-1** — start of `solve()`, before reading input. Stepped through the input parsing: `n`, `m`, `h`, the horse array, and the adjacency list construction. Checked `horses` and `adj` were built correctly after the loops. They were. Ruled out a parsing or off-by-one problem in the graph build.

**BP-2** — after both `dijkstra()` calls returned, before the answer loop. Inspected `d1` and `dn`. The distances were correct. This confirmed what the verdict already implied (the algorithm is right) and narrowed the problem to "correct result, wrong cost", i.e. something inside the Dijkstra that does redundant work.

**The obstacle.** The Dijkstra was written as a lambda (`auto dijkstra = [&]`), and stepping through it was awkward — I could see it from the outside but not navigate its body the way I wanted. I stopped fighting the tool and restructured instead: pulled the lambda body out as plain inline code inside `solve()`, then re-ran under the debugger and stepped line by line through the relaxation.

**The find.** Stepping into the edge relaxation, the loop ran far more times than the degree of the node. Looking at the line I had stopped on:

```cpp
for (auto &[to, w] : adj[node]) {
    for (auto &[to, w] : adj[node]) {   // same loop, written twice
        ll nw = hasHorse ? w / 2 : w;
        ...
    }
}
```

Two nested identical range-for loops over `adj[node]`. The inner structured binding `[to, w]` shadows the outer one, so the outer loop's variables are never read — it just executes the inner loop `deg(node)` times.

## Why this produced TLE and not WA

Dijkstra's relaxation step is idempotent. Once a node is popped and its neighbours relaxed, running the same relaxation again changes nothing, because no further improvement is possible. So the duplicate loop was pure wasted work that left every distance correct.

Cost per pop went from O(deg) to O(deg²), making the whole run O(sum of deg²) instead of O(m log n). On a star-shaped graph with m = 2_10^5 that is roughly 4_10^10 iterations — asymptotically worse, not a constant factor.

## The fix

Removed the outer loop. While rewriting I also flattened the state representation: `vector<array<ll,2>> dist` became a flat `vector<ll> dist(2*n)` indexed by `2*node + hasHorse`, and the priority queue went from `tuple<ll,int,int>` to `pair<ll,int>` with the state bit-packed into the second field. Same algorithm and same 2n state space, but a flatter memory layout and a cheaper heap comparison (two fields instead of three).

Worth being precise about the relative weight of these two changes: the flat layout and the smaller tuple are a constant-factor improvement, maybe under 2x. Deleting the nested loop is what changed the complexity class and is what made the submission pass. It would have been easy to credit the memory rework for the fix, and it would have been wrong.

