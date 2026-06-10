# Flywire Internship Technical Assessment

This repository is my submission for the summer 2026 FlyWire internship. If you're looking for my biological analysis of the final circuit, see `science.md`.

My process in solving this problem is composed of four different approaches, increasing in complexity as the true difficulty of the problem became increasingly clear.

## Approach 1: Naive ID Matching

Looking back on Approach 1, I can see how I fell into this rabbit hole. When initially scanning the datasets, I noticed that the male fly datasets used 5-digit IDs, and checking for overlap produced thousands of results. Thus, I wrongfully concluded that I could perform simple `pandas` merges on the matching IDs in the MAOL, MCNS, and MANC edgelists, then identify the largest connected component. Doing this, I got a 798-node "subgraph".

The error was assuming that IDs correspond to the same neurons in different datasets. I quickly realized when attempting to address why I didn't use the female fly datasets (which use 18-digit IDs) that this approach doesn't really make any sense - female flies should still have the same circuitry even if their ID numbers are different, so there's no reason to think that ID numbers actually mean anything. A quick check on Codex verified that one ID number corresponded to three completely different neurons in MAOL, MCNS, and MANC.

## Approach 2: Naive Fingerprint Matching

The second approach involved using degree fingerprints - the in-degree and out-degree pair for each neuron - to try and match homologous neurons. 

First, to increase the likelihood of finding biologically homologous neurons, I replaced MANC with FAFB so that all three datasets had optic lobes in their connectomes. My hope was that I would find a circuit pertaining to visual perception, particularly some circuitry for photoreception rather than something like image processing or motion detection.

To try and filter out false positives from neurons that could share the same fingerprint without being biologically homologous, I first filtered each dataset for nodes with unique fingerprints - in essence, I was trying to filter for hubs that would match up across the datasets while increasing the likelihood that they perform the same functions in the same circuits. Doing this, I got 346 candidate neurons that had matching fingerprints across all three datasets. My plan was to then build outward from those candidates - check the fingerprints of each candidate's neighbors, keep the candidates that still matched, and continue expanding until only one subgraph was identified.

The challenge, however, was trying to expand to just the candidates' immediate neighbors. Just for a single hop, all candidates were eliminated. Not one solution came out of this method.

It all came down to the densitity of the datasets. MAOL, while having much fewer nodes than FAFB (52k vs. 140k), is much more synaptically dense (6.5M edges vs 3.7M). Thus, each candidate in MAOL is going to have far more neighbors than FAFB or MCNS, and those neighbors' fingerprints are also going to be substantially different. Hardly any were going to match exactly, and the ones that did would basically never have neighbors that matched as well.

I knew when starting this approach that it would be unlikely for there to be that many candidates using exact fingerprint matching, but I didn't anticipate how much denser MAOL would be than FAFB and MCNS. Looking back on it now, this approach was also kind of doomed frmo the start.

## Approach 3: Fingerprint Meta-Graph

The third approach was a modified version of Approach 2 where I essentially replaced neuron IDs with fingerprints. Then, I merged the original edge lists using the fingerprints (with some pre-filtering). This is kind of a mix between Approaches 1 and 2. Earlier, I was trying to first find candidates, then build outward from each candidate. Instead, I simply treated the fingerprints as new "names" for the nodes, then merged based on the shared "names". This, of course, comes at the cost of being really unrefined. Now we're ensuring fingerprints are well matched, but since one fingerprint could correspond to multiple nodes, it was hard to be able to find actually corresponding neurons.

I was able to get a meta-graph with 5,887 edges, but the entire thing was one connected component so there was no way to refine it further. It didn't make sense to extract the IDs for each respective dataset and then calculate the largest connected components either, as you'd get different results for each dataset (e.g., I got a 68-node connected component in MAOL but only a 9-node connected component in FAFB).

## Approacch 4: Conflict and Compatibility Graphs

The last Hail Mary I threw leveraged conflict graphs - a graph in which edges are drawn between node gropus (two groups of corresponding nodes) if there is any difference in edges between each pair of nodes between each group. Applied to these datasets, I created a graph with each node being candidate correspondents - triplets of one MAOL neuron, one FAFB neuron, and one MCNS neuron - then drew edges between nodes if the connectivity of the MAOL neuron pair conflicted with that of the FAFB or MCNS pairs. Once the conflict graph is built, I found the maximum independent set, i.e. the largest set of triplets that have no conflict edges drawn between them.

I tried using the 346 candidates with unique fingerprints to start with, but the resulting 128 surviving triplets produced an empty graph (zero edges). Technically passes the requirements for a conflicting graph, but gives me nothing for biological analysis. Considering nodes with no connection to be conflicting drew edges between every triplet - also not useful.

The issue was that we were still doing exact matching. But if a hub is conserved across datasets, it should still have the same degree profile *relative to the rest of the dataset*. Therefore, we tried using two relative measures - the directional ratio and the degree rank percentile. By rounding floating point numbers to 3 decimals, we got 34k triplets. We then filtered for nodes that had at least one edge between them, reducing the candidate list down to 1,882 triplets. However, the conflict graph still maxed out, and the compatibility graph produced zero edges.

Including a tolerance for directional ratio (0.02) and degree rank percentile (0.15) produced a largest clique of 3 but a largest connected component of 455. However, the unique number of neurons from each dataset still differed. We then did a filtering step that ranks the triplets by connectivity and skips past any triplets in which a neuron has already matched in an ealier triplet. This brought it down to 157 nodes.

However, the verification script that ensured my answer was correct failed. There were still inconsistent edges.

The clique was also shown to be incorrect as FAFB had two edges going between each node, while MAOL and MCNS each had one edge.