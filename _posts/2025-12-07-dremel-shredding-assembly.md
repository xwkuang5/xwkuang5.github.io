---
layout: post
title: "Deep Dive into Dremel: Record Shredding and Assembly"
date: 2025-12-07
categories: [Data Engineering, Algorithms]
published: true
---

In the world of big data, efficient storage and retrieval of nested data structures are paramount. [Google's Dremel system](https://research.google/pubs/dremel-interactive-analysis-of-web-scale-datasets/) (the precursor to BigQuery and the inspiration for the popular Parquet format) introduced a novel columnar storage format that handles nested data (like JSON or Protocol Buffers) with remarkable scanning efficiency. In this post, I'll share my own implementation of the Dremel record shredding and assembly algorithms, with code snippets and explanations of the core concepts. The implementation mostly follows the pseudocode as described in the original paper so you may also find it useful to read the original paper for more details.

## 1. High-Level Overview

Dremel's core innovation is its ability to **shred** (decompose) nested records into separate columns while preserving the structural information needed to **assemble** (reconstruct) them. This columnar format allows for:

- **IO Efficiency**: You only read the columns you need for a query.
- **Better Compression**: Data of the same type is stored together, leading to higher compression ratios.
- **Fast Querying**: Vectorized operations can be applied to columns directly.

The core mechanism relies on two metadata values stored with each datum: **Repetition Level (r)** and **Definition Level (d)**. These levels encode the structural history of each value, allowing us to flatten the tree structure into simple lists of values.

## 2. Setup and Assumptions

In my implementation, I made a few simplifying assumptions to focus on the core algorithms:

- **Input Format**: Records are provided as a JSON array of JSON objects.
- **Schema Format**: I use a path-based schema definition where `[*]` denotes a repeated field.

For example, a schema might look like this:

```python
schema_paths = [
    "doc.links[*].url",
    "doc.links[*].language"
]
root = parse_schema(schema_paths)
```

Internally, this is parsed into a tree of `ColumnDescriptor` objects, each tracking its `max_repetition_level` and `max_definition_level`.

## 3. The Shredding Algorithm

Shredding involves traversing the record and writing values to their respective columns along with their repetition and definition levels.

### Pseudocode

Here's a simplified view of the shredding logic:

```python
def dissect_record(decoder, writer, repetition_level):
    seen_fields = set()

    while decoder.has_next():
        field, value = decoder.next()
        child_writer = writer.get_child(field)

        # Calculate definition level
        definition_level = decoder.definition_level + 1

        if child_writer.is_repeated:
            for i, item in enumerate(value):
                # First item inherits current repetition level
                # Subsequent items use the field's max repetition level
                child_r = repetition_level if i == 0 else child_writer.max_repetition_level

                if child_writer.is_leaf():
                    child_writer.write(item, child_r, definition_level)
                else:
                    dissect_record(RecordDecoder(item, definition_level),
                                 child_writer, child_r)
        else:
            # Handle optional fields (similar to repeated fields)...
            pass

    # Write nulls for missing fields
    for field, child_writer in writer.children.items():
        if field not in seen_fields:
            write_nulls(child_writer, repetition_level, decoder.definition_level)
```

## 4. The Finite State Machine (FSM)

To reconstruct records efficiently, Dremel constructs a Finite State Machine. The FSM tells us, given the current column and the next repetition level, which column to read next.

The FSM construction involves:

1.  **Barrier Edges**: Moving to the next column in the schema.
2.  **Back Edges**: Jumping back to a previous column when a repetition occurs.

```python
def make_fsm(schema):
    fsm = defaultdict(dict)
    fields = list(get_leaves(schema))

    for index, field in enumerate(fields):
        # Determine the "barrier": the next field in the schema
        barrier = fields[index + 1] if index < len(fields) - 1 else END
        barrier_level = common_ancestor(field, barrier).max_repetition_level if barrier != END else 0

        # Step 1: Add back edges
        # If we see a repetition level that belongs to a common ancestor with a previous field,
        # we might need to jump back.
        for pre_field in reversed(fields[:index]):
            if pre_field.max_repetition_level <= barrier_level:
                continue
            back_level = common_ancestor(pre_field, field).max_repetition_level
            fsm[field][back_level] = pre_field

        # Step 2: Fill gaps
        # Ensure that for every possible repetition level, we have a transition.
        # If a level isn't explicitly handled, it usually implies continuing with the current field
        # or bubbling up to a higher level handler.
        for level in reversed(range(barrier_level + 1, field.max_repetition_level + 1)):
            if level not in fsm[field]:
                fsm[field][level] = field if level == field.max_repetition_level else fsm[field][level + 1]

        # Step 3: Add barrier edges
        # For lower repetition levels (indicating we are done with the current repeated group),
        # we move to the barrier (next field).
        for level in range(barrier_level + 1):
            fsm[field][level] = barrier

    return fsm
```

## 5. The Assembly Algorithm

Assembly is the reverse process of shredding. We assemble records by reading values from multiple columns, using the FSM to switch between them.

### Pseudocode

The core assembly loop looks something like this:

```python
def assemble_record(fsm, readers, assemblers):
    descriptor = readers[0].descriptor
    assembler = Assembler(root_descriptor)

    while descriptor != END:
        # 1. Move to the correct nesting level for this field
        #    (opens nested records if needed)
        assembler.move_to_level(descriptor.max_definition_level, descriptor)

        # 2. Read the next value
        reader = readers[descriptor]
        value, r, d = reader.next()

        # 3. Add value to record if present
        if d == descriptor.max_definition_level:
            assemblers[descriptor].add(value, assembler)

        # 4. Determine next field using FSM
        next_repetition_level = reader.peek().repetition_level if reader.has_next() else 0
        next_descriptor = fsm[descriptor][next_repetition_level]

        # 5. Handle repetition (closing scopes)
        if next_descriptor != END and is_repeating(descriptor, next_descriptor):
             target_def_level = descriptor.full_repetition_level(next_repetition_level)
             assembler.return_to_level(target_def_level)

        descriptor = next_descriptor

    assembler.return_to_level(0)
    return assembler.buffer
```

### Handling Record Structure with `return_to_level`

A crucial detail in assembly is knowing when to _close_ nested records. The FSM guides us between columns, but it doesn't explicitly tell us when to exit a nested object.

For example, if we see a repetition level that implies we are repeating `doc.links[*]`, we might transition from `doc.links[*].url` back to `doc.links[*].url`. However, we need to know that this transition implies starting a **new** `links` object, rather than just adding another `url` to the existing one (if `url` itself were repeated).

This is where `return_to_level` comes in. We use the `next_repetition_level` to determine the "full repetition level" — the definition level of the node that is actually repeating.

```python
# In assembly.py

next_repetition_level = reader.peek()[1] if reader.has_next() else 0
next_descriptor = fsm[descriptor][next_repetition_level]

if next_descriptor is not END and assembler.is_repeating(descriptor, next_descriptor):
    # This is the critical step!
    # We find the definition level corresponding to the repetition
    target_def_level = descriptor.full_repetition_level(next_repetition_level)
    assembler.return_to_level(target_def_level)
```

By calling `return_to_level`, the assembler closes all open scopes (nested dicts/lists) deeper than the repeating node, effectively preparing the buffer for the new repetition.

## Demo

I've created a Streamlit app that allows you to play around with the shredding and assembly process. You can try it out here ([![Streamlit App](https://static.streamlit.io/badges/streamlit_badge_black_white.svg)](https://is4mel7ushhrrrcfnpgxey.streamlit.app/)).

The source code for the implementation and the demo app is available on GitHub: [xwkuang5/dremel](https://github.com/xwkuang5/dremel).

## 6. Implementation Nuances

### Lazy Writing for Sparse Values

Dremel is designed for sparse datasets where a record may have thousands of fields, but only a handful are populated in any given instance. In such cases, Dremel writes NULL values lazily. It tracks the repetition and definition levels of NULLs seen so far in the ancestors of leaf fields, and only invokes the writer for a leaf field when there is a non-null value to write or when the shredding session is complete.

**Note:** My current implementation does not do this yet and **eagerly** writes all leaves for simplicity.

### Lossless Encoding of Absent vs. Empty

Depending on the use case, some applications may require distinguishing between an _absent_ sub-record and an _empty_ sub-record (present but empty). For example, `{}` vs `{"links": []}`. Dremel can handle this by adding **explicit columns stripes for non-leaf fields**. We can use a column stripe to record the existence or absence of a sub-record using the definition level.

**Note:** My current implementation does not do this yet.

---

Understanding Dremel's internals gives us a profound appreciation for modern columnar stores. The separation of structure from data allows for incredibly flexible and efficient data processing.
