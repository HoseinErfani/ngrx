```python


import os
import re
from collections import defaultdict, deque

# Function to find a component's selector from its TypeScript file
def find_component_selector(file_path):
    with open(file_path, 'r') as f:
        content = f.read()
    match = re.search(r'@Component\(\{[^}]*selector:\s*[\'"]([^\'"]+)[\'"]', content)
    return match.group(1) if match else None

# Recursive function to track method usage to the end component/page
def track_method_usages(method_details, method_usages, target_method, angular_dir):
    end_components = set()
    queue = deque([{'method': target_method, 'file': method_details[target_method]['file']}])

    while queue:
        current = queue.popleft()
        current_method = current['method']
        current_file = current['file']

        if current_method in method_usages:
            for usage_file in method_usages[current_method]:
                if "component.ts" in usage_file:
                    selector = find_component_selector(usage_file)
                    if selector:
                        # Check if this selector is used in any HTML
                        if not search_selector_in_html(angular_dir, selector, True):
                            queue.append({'method': selector, 'file': usage_file})
                        else:
                            end_components.add((usage_file, "PAGE"))
                    else:
                        end_components.add((usage_file, "PAGE"))
        else:
            # Add the file as an end component if no further usage is found
            end_components.add((current_file, "PAGE"))

    return end_components

# Function to search for a selector in HTML files, optionally recursive
def search_selector_in_html(dir, selector, recursive=False):
    files = os.listdir(dir)
    for file in files:
        file_path = os.path.join(dir, file)

        if os.path.isfile(file_path) and file_path.endswith('.html'):
            with open(file_path, 'r') as f:
                content = f.read()
                if re.search(r'<{}\b'.format(re.escape(selector)), content):
                    return True
        elif os.path.isdir(file_path) and recursive:
            if search_selector_in_html(file_path, selector, recursive):
                return True
    return False

# Function to find variable values within a class
def find_variable_value_in_class(content, variable_name):
    pattern = r'\b{}\s*:\s*\w+\s*=\s*[\'"]([^\'"]+)[\'"]'.format(re.escape(variable_name))
    match = re.search(pattern, content)
    if match:
        return match.group(1)
    return None

# Function to find all methods using apiService.get/post and their details
def find_api_methods(content):
    method_pattern = r'(\w+)\s*\(.*?\)\s*:\s*Observable<[^>]+>\s*{\s*return\s+this\.apiService\.(get|post)\((["`])([^\'"`]+)\3'
    return re.findall(method_pattern, content)

# Function to find method usages in files
def find_method_usages(dir, method_names):
    method_usage = defaultdict(set)

    files = os.listdir(dir)
    for file in files:
        file_path = os.path.join(dir, file)

        if os.path.isfile(file_path) and file_path.endswith('.ts'):
            with open(file_path, 'r') as f:
                content = f.read()

            for method in method_names:
                if re.search(r'\b{}\b'.format(re.escape(method)), content):
                    method_usage[method].add(file_path)
        elif os.path.isdir(file_path):
            nested_usages = find_method_usages(file_path, method_names)
            for key, value in nested_usages.items():
                method_usage[key].update(value)

    return method_usage

# Function to find component path in route module files
def find_component_path_in_routes(component_name, dir):
    files = os.listdir(dir)
    for file in files:
        file_path = os.path.join(dir, file)

        if os.path.isfile(file_path) and file.endswith('-routing.module.ts'):
            with open(file_path, 'r') as f:
                list_started = False
                path_dict = {}
                for line in f:
                    if list_started:
                        trimmed_line = line.strip()
                        if trimmed_line.startswith('{'):
                            path_dict = {}
                        elif trimmed_line.startswith('path:'):
                            regex = r'^path:\s*[\'"](.*)[\'"]\s*,$'
                            match = re.search(regex, line)
                            path_dict['path'] = match.group(1)
                        elif trimmed_line.startswith('component:'):
                            print('test')
                        elif trimmed_line.startswith('}'):
                            print(path_dict)
                    else:
                        trimmed_line = line.strip()
                        if trimmed_line.endswith('['):
                            list_started = True
                # content = f.read()
                # pattern = r'path:\s*[\'"]([^\'"]+)[\'"]\s*,\s*component:\s*' + component_name + r'\b'
                # match = re.search(pattern, content)
                # print "match {} ".format(match)
        elif os.path.isdir(file_path):
            result = find_component_path_in_routes(component_name, file_path)
            if result:
                return result
    return None

# Main function to search files
def search_in_files(dir):
    result = defaultdict(list)
    method_details = {}

    files = os.listdir(dir)
    for file in files:
        file_path = os.path.join(dir, file)

        if os.path.isfile(file_path) and file_path.endswith('.ts'):
            with open(file_path, 'r') as f:
                content = f.read()

            methods = find_api_methods(content)
            for method, http_method, _, url in methods:
                url = re.sub(r'\${this\.(\w+)}',
                             lambda m: find_variable_value_in_class(content, m.group(1)) or m.group(0),
                             url)

                result[file_path].append({
                    'method': method,
                    'http_method': http_method,
                    'url': url,
                })
                method_details[method] = {'file': file_path, 'url': url}

        elif os.path.isdir(file_path):
            nested_results, nested_method_details = search_in_files(file_path)
            for key, value in nested_results.items():
                result[key].extend(value)
            method_details.update(nested_method_details)

    return result, method_details

# Execute search in the Angular project directory
angular_dir = os.path.join(os.path.dirname(__file__), 'src/app')
results, method_details = search_in_files(angular_dir)

method_usages = find_method_usages(angular_dir, method_details.keys())
final_results = {}

for method, details in method_details.items():
    end_components = track_method_usages(method_details, method_usages, method, angular_dir)
    component_results = []

    for component_file, status in end_components:
        if status == "PAGE":
            component_name_match = re.search(r'export\s+class\s+(\w+)\s+', open(component_file).read())
            component_name = component_name_match.group(1) if component_name_match else None
            if component_name:
                route_path = find_component_path_in_routes(component_name, angular_dir)
                component_results.append((component_name, route_path))

    final_results[method] = {
        'url': details['url'],
        'components': component_results
    }

# Display results
if final_results:
    for method, details in final_results.items():
        print "URL: {}".format(details['url'])
        print "USAGE:"
        for component, path in details['components']:
            print "  component: {}".format(component)
            print "  path: \"{}\"".format(path)
        print '\n--------------------------------------------------------------\n'
else:
    print 'No apiService calls found.'

```
