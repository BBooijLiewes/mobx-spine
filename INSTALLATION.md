# Installing the Memory Efficiency Branch

You can use the memory efficiency improvements branch directly in your project before it's merged to master.

## Option 1: Install from GitHub Branch (Recommended)

Update your `package.json` to point to the GitHub branch:

```json
{
  "dependencies": {
    "mobx-spine": "BBooijLiewes/mobx-spine#feature/memory-efficiency-improvements"
  }
}
```

Then run:
```bash
npm install
# or
yarn install
```

## Option 2: Install from GitHub Branch with Specific Commit

For more stability, you can pin to a specific commit:

```json
{
  "dependencies": {
    "mobx-spine": "BBooijLiewes/mobx-spine#2fab4b4"
  }
}
```

## Option 3: Install from Git URL

```json
{
  "dependencies": {
    "mobx-spine": "git+https://github.com/BBooijLiewes/mobx-spine.git#feature/memory-efficiency-improvements"
  }
}
```

## Option 4: Install with npm/yarn directly

```bash
npm install BBooijLiewes/mobx-spine#feature/memory-efficiency-improvements
# or
yarn add BBooijLiewes/mobx-spine#feature/memory-efficiency-improvements
```

## Verification

After installation, verify you have the correct version:

```javascript
// In your code, check if dispose method exists
import { Model, Store } from 'mobx-spine';

const store = new Store();
console.log(typeof store.dispose); // Should output: "function"
```

## Switching Back to Official Release

When the PR is merged and a new version is published, update your `package.json` back to:

```json
{
  "dependencies": {
    "mobx-spine": "^0.28.7"
  }
}
```

(Or whatever the next version number will be)

## Notes

- The branch includes all the memory improvements and is fully tested
- All 251 tests pass
- 100% backward compatible with existing code
- No code changes required in your application
- To benefit from dispose pattern, add cleanup calls in your component lifecycle methods

## Example Usage in Your Project

```javascript
import { Model, Store } from 'mobx-spine';

class MyStore extends Store {
  // Your store code
}

// In your component
class MyComponent extends React.Component {
  store = new MyStore();

  componentWillUnmount() {
    // Clean up to prevent memory leaks
    this.store.dispose();
  }

  render() {
    // Your component code
  }
}
```
