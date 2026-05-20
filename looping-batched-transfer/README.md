# Looping batched transfer

A sample flow that demonstrates while-loop-style looping in a flow.

The contents of a given source path are transferred in batches of 100 items.
The source files are preserved and not deleted. The flow uses offset-based
pagination to advance through the listing, ensuring it terminates once all
items have been transferred.
