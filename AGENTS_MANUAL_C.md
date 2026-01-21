# Linux Kernel C Coding Manual & Agent Directives

This document encapsulates the coding standards, architectural patterns, and "brilliant logic" of the Linux Kernel. **It is specifically optimized as a directive set for AI Agents and Developers.**

## 1. Core Directives for Agents

**Objective:** You are an expert Kernel Hacker. Your code must be robust, performant, and maintainable. You value pragmatism over purity.

### 1.1 The "Verify Before You Hallucinate" Rule
*   **Problem:** LLMs often invent APIs.
*   **Directive:** Before using a kernel function (e.g., `kmalloc`, `list_add_rcu`), **verify its existence** in the codebase (specifically in `include/linux/`).
*   **Action:** If you are unsure if `kfree_sensitive` exists vs `kzfree`, search the headers first.

### 1.2 The "No Floating Code" Rule
*   **Directive:** Every piece of code must be aware of its execution context.
*   **Check:** Are you in an interrupt context? (Cannot sleep). Are you holding a spinlock? (Cannot sleep).
*   **Action:** If writing a driver probe function, explicitly check if `GFP_KERNEL` (can sleep) or `GFP_ATOMIC` (cannot sleep) is appropriate.

### 1.3 Strict Formatting (Torvalds' Law)
*   **Indentation:** Hard tabs, 8 characters wide. **Do not use spaces for indentation.**
*   **Braces:** Open brace on the same line for control structures (`if`, `for`). Open brace on a new line for functions.
*   **Line Length:** 80 columns preferred, but do not break lines if it hurts readability.

## 2. Architecture & Organization

The kernel is monolithic but modular.

*   **`arch/`**: Hardware-specific code. Do not touch unless explicitly targeting a specific CPU.
*   **`drivers/`**: Where 70% of the code lives. Focus your efforts here for device support.
*   **`include/linux/`**: The "API Reference". Read this to understand available structures.
*   **`mm/`**: Memory management guts. High complexity/risk.

## 3. Best Practices & Patterns

### 3.1 Centralized Exit Paths (The "Good" Goto)
**Rule:** Use `goto` to manage error handling and resource cleanup. It prevents nested `if` spaghettification.

**Agent Template:**
```c
int my_driver_probe(struct platform_device *pdev)
{
    int ret;
    struct my_data *data;

    data = kzalloc(sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    ret = some_init_function();
    if (ret)
        goto err_free_mem; /* Jump to cleanup */

    return 0;

err_free_mem:
    kfree(data);
    return ret;
}
```

### 3.2 Intrusive Linked Lists (`struct list_head`)
**Concept:** The data contains the list node, not vice-versa.
**Directive:** Never invent your own list structure. Use `include/linux/list.h`.

**The "Magic" Macro: `container_of`**
*   **Usage:** To get the parent structure from a list node.
*   **Agent Logic:** If you have a `struct list_head *ptr`, finding the container requires knowing the type and the member name.
    ```c
    struct my_struct *item = container_of(ptr, struct my_struct, list_node);
    ```

### 3.3 Branch Prediction Optimization
**Directive:** If an error condition is rare (it should be), tell the compiler.
*   `if (unlikely(ret < 0))` -> Moves the error handling code out of the hot path.
*   `if (likely(valid_packet))` -> Keeps this path in the instruction cache.

### 3.4 Compile-Time Assertions
**Directive:** Catch errors at build time, not runtime.
*   **Use:** `BUILD_BUG_ON(sizeof(struct my_hw_struct) != 128);`
*   **Why:** Ensures hardware alignment requirements are met before the code ever runs.

## 4. Memory Management Directives

### 4.1 No Garbage Collection
**Directive:** You are the garbage collector. Every `kmalloc` must have a corresponding path to `kfree`.

### 4.2 Reference Counting
**Directive:** If a structure is used by multiple threads/contexts, use `refcount_t`.
*   **Pattern:** `kref_get` / `kref_put`.
*   **Cleanup:** The release function is called only when the count hits zero.

### 4.3 RCU (Read-Copy-Update)
**Context:** High-performance read-heavy scenarios.
**Directive:**
1.  **Readers:** `rcu_read_lock()` ... `rcu_read_unlock()`. Zero overhead.
2.  **Writers:** Copy data -> Modify copy -> Switch pointer -> `synchronize_rcu()` (wait for readers).
3.  **Agent Check:** Never sleep inside an RCU read critical section.

## 5. Agent Workflow Summary

1.  **Analyze Context:** Read `AGENTS.md` (if present) and `include/` files relevant to the task.
2.  **Plan:** Determine which subsystem (`drivers/net`, `fs/`, etc.) you are modifying.
3.  **Code:**
    *   Use `snake_case`.
    *   Use `goto` for error handling.
    *   Use `likely`/`unlikely` for optimization.
4.  **Verify:** Check for "sleeping in atomic context" bugs. Verify `kmalloc` return values.
