# Memory Profiling Guide for mobx-spine

If you're not seeing expected memory improvements, here's how to diagnose and optimize your specific use case.

## Quick Diagnosis Questions

1. **Are you calling `dispose()`?**
   - The memory improvements require explicit cleanup
   - Without calling `dispose()`, memory leaks will persist

2. **Do you use virtual stores?**
   - Virtual stores create autoruns that must be disposed
   - Check if you're cleaning them up

3. **Do you have file uploads?**
   - Blob URL leaks only occur with file uploads
   - Check if you're using `setInput()` with files

4. **How many models do you have?**
   - Shallow observable benefits scale with model count
   - Benefits are minimal with <50 models

## Step 1: Verify You Have the Right Version

```javascript
import { Store, Model } from 'mobx-spine';

const store = new Store();
const model = new Model();

console.log('Has dispose:', typeof store.dispose === 'function');
console.log('Has __limit:', '__limit' in store);
console.log('Has __blobUrls:', '__blobUrls' in model);

// All should be true
```

## Step 2: Profile Your Application

### Chrome DevTools Memory Profiling

1. Open Chrome DevTools → Memory tab
2. Take a heap snapshot (baseline)
3. Use your application normally
4. Take another heap snapshot
5. Compare snapshots

**Look for:**
- Growing arrays of models/stores
- Unreleased blob URLs (search for "blob:")
- Detached DOM nodes with model references
- Growing `__changes` arrays

### Check for Common Issues

```javascript
// Issue 1: Not disposing stores
class MyComponent {
    store = new MyStore();
    
    componentWillUnmount() {
        // ❌ MISSING: this.store.dispose();
    }
}

// Issue 2: Not disposing virtual stores
const virtualStore = store.virtualStore({ filter: x => x.active });
// ❌ MISSING: virtualStore.dispose() when done

// Issue 3: Creating many models without cleanup
models.forEach(data => {
    const model = new MyModel(data);
    // ❌ MISSING: model.dispose() when done
});

// Issue 4: Keeping references to old models
this.oldModels.push(model); // ❌ Prevents GC even after dispose
```

## Step 3: Measure Memory Usage

### Before Optimization

```javascript
// Add this to your app
class MemoryMonitor {
    constructor() {
        this.measurements = [];
    }
    
    measure(label) {
        if (performance.memory) {
            this.measurements.push({
                label,
                usedJSHeapSize: performance.memory.usedJSHeapSize,
                totalJSHeapSize: performance.memory.totalJSHeapSize,
                timestamp: Date.now()
            });
            console.log(`[${label}] Memory: ${
                (performance.memory.usedJSHeapSize / 1048576).toFixed(2)
            } MB`);
        }
    }
    
    report() {
        console.table(this.measurements);
    }
}

// Usage
const monitor = new MemoryMonitor();

monitor.measure('Initial');
// ... load data
monitor.measure('After Load');
// ... use app
monitor.measure('After Use');
// ... cleanup
monitor.measure('After Cleanup');

monitor.report();
```

### After Adding Dispose Calls

```javascript
class MyComponent {
    store = new MyStore();
    
    componentDidMount() {
        this.store.fetch();
    }
    
    componentWillUnmount() {
        // ✅ Proper cleanup
        this.store.dispose();
        this.store = null;
    }
}
```

## Step 4: Identify Your Bottleneck

### Memory Leak Patterns

#### Pattern 1: Autorun Leaks (Virtual Stores)
```javascript
// ❌ BAD: Creates autorun that never stops
const filtered = store.virtualStore({ filter: x => x.active });

// ✅ GOOD: Dispose when done
const filtered = store.virtualStore({ filter: x => x.active });
// Later:
filtered.dispose();
```

#### Pattern 2: Blob URL Leaks (File Uploads)
```javascript
// ❌ BAD: Old code creates blob URLs that persist
model.setInput('avatar', fileObject);
// Blob URL created but never revoked

// ✅ GOOD: New code automatically revokes
model.setInput('avatar', fileObject);
// Old blob URL automatically revoked
model.dispose(); // Revokes all remaining blob URLs
```

#### Pattern 3: Circular References
```javascript
// ❌ BAD: Circular references prevent GC
model.store = store;
store.models.push(model);
// Even after removing from store, model.store keeps it alive

// ✅ GOOD: Use dispose to break cycles
store.dispose(); // Clears models array
model.dispose(); // Clears relations
```

#### Pattern 4: Event Listeners
```javascript
// ❌ BAD: MobX reactions not cleaned up
class MyComponent {
    constructor() {
        this.store = new MyStore();
        autorun(() => {
            console.log(this.store.models.length);
        }); // Never disposed!
    }
}

// ✅ GOOD: Track and dispose reactions
class MyComponent {
    constructor() {
        this.store = new MyStore();
        this.disposer = autorun(() => {
            console.log(this.store.models.length);
        });
    }
    
    cleanup() {
        this.disposer();
        this.store.dispose();
    }
}
```

## Step 5: Expected Memory Savings by Use Case

### Use Case 1: Simple CRUD App (50-100 models)
- **Expected savings:** 5-10%
- **Why:** Shallow observables help, but overhead is already low
- **Key benefit:** Proper GC when navigating away

### Use Case 2: Large Dataset (500+ models)
- **Expected savings:** 15-25%
- **Why:** Shallow observables scale with model count
- **Key benefit:** Faster updates, less memory per model

### Use Case 3: File Upload Heavy
- **Expected savings:** 20-40%
- **Why:** Blob URLs can accumulate significantly
- **Key benefit:** Prevents memory leaks over time

### Use Case 4: Virtual Stores / Filtered Views
- **Expected savings:** 30-50%
- **Why:** Autorun leaks are severe
- **Key benefit:** Prevents memory growth over time

### Use Case 5: Long-Running SPA (hours of use)
- **Expected savings:** 40-60% over time
- **Why:** Prevents accumulation of leaks
- **Key benefit:** Stable memory usage over time

## Step 6: Optimization Checklist

- [ ] Verified correct version installed
- [ ] Added `dispose()` calls to all stores in component cleanup
- [ ] Added `dispose()` calls to all virtual stores
- [ ] Added `dispose()` calls to standalone models
- [ ] Removed references to disposed objects
- [ ] Checked for custom autoruns/reactions and disposed them
- [ ] Profiled with Chrome DevTools
- [ ] Measured memory before/after
- [ ] Checked for growing arrays in heap snapshots

## Step 7: If Memory Still High

If you've done all the above and memory is still high, the issue might be:

### 1. **Not a mobx-spine Issue**
```javascript
// Check what's using memory
// In Chrome DevTools → Memory → Take Heap Snapshot
// Sort by "Retained Size"
// Look for:
// - Large arrays not related to mobx-spine
// - DOM nodes
// - Images/media
// - Third-party libraries
```

### 2. **Need Additional Optimizations**
The current PR focuses on memory leaks. For performance/memory with large datasets, you might need:

- Virtualization (only render visible models)
- Pagination (don't load all data at once)
- Lazy loading (load relations on demand)
- Data pruning (remove old models from stores)

### 3. **MobX Configuration**
```javascript
import { configure } from 'mobx';

// Stricter mode can help identify issues
configure({
    enforceActions: 'always',
    computedRequiresReaction: true,
    reactionRequiresObservable: true,
    observableRequiresReaction: true,
});
```

## Example: Full Cleanup Implementation

```javascript
import React from 'react';
import { observer } from 'mobx-react';
import { MyStore } from './stores';

@observer
class MyComponent extends React.Component {
    constructor(props) {
        super(props);
        this.store = new MyStore();
        this.disposers = [];
    }
    
    componentDidMount() {
        // Load data
        this.store.fetch();
        
        // If you have custom reactions
        this.disposers.push(
            autorun(() => {
                console.log('Models:', this.store.models.length);
            })
        );
    }
    
    componentWillUnmount() {
        // Clean up reactions
        this.disposers.forEach(d => d());
        this.disposers = [];
        
        // Clean up store
        this.store.dispose();
        this.store = null;
    }
    
    render() {
        return (
            <div>
                {this.store.models.map(model => (
                    <div key={model.id}>{model.name}</div>
                ))}
            </div>
        );
    }
}

export default MyComponent;
```

## Need Help?

If you're still experiencing high memory usage:

1. Share your heap snapshot analysis
2. Describe your use case (model count, relations, file uploads)
3. Show your cleanup code
4. Provide memory measurements before/after

This will help identify if you need:
- Better cleanup implementation
- Additional optimizations (see ADDITIONAL_OPTIMIZATIONS.md)
- Different architectural approach
