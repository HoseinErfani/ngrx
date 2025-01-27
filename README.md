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

def track_method_usages_v2(original_method_details, dir):
    method_details = original_method_details.copy()
    end_components = set()
    flag = True
    component_selectors = method_details.keys()
    
    while flag:
        component_usages = search_selector_in_html(dir, component_selectors, True)
        used_component_selectors = component_usages.keys()
        for selector in component_selectors:
            if used_component_selectors.__contains__(selector):
                usages = component_usages[selector]
                new_selectors = []
                for usage in usages:
                    print(usage)
            else:
                end_components.add(method_details[selector]['file'])

# Recursive function to track method usage to the end component/page
def track_method_usages(method_details, method_usages, target_method, angular_dir):
    end_components = set()
    queue = deque([{'method': target_method, 'file': method_details[target_method]['file']}])
    print('target_method : {} '.format(target_method))
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
def search_selector_in_html(dir, selectors, recursive=False):
    files = os.listdir(dir)
    selector_usages = {}
    for file in files:
        file_path = os.path.join(dir, file)

        if os.path.isfile(file_path) and file_path.endswith('.html'):
            with open(file_path, 'r') as f:
                content = f.read()
                for selector in selectors:
                    if re.search(r'<{}\b'.format(re.escape(selector)), content):
                        if selector_usages.__contains__(selector):
                            selector_usages[selector].append(file_path)
                        else:
                            selector_usages[selector] = [].append(file_path)
        elif os.path.isdir(file_path) and recursive:
            internal_selector_usages = search_selector_in_html(file_path, selectors, recursive)
            for key, value in internal_selector_usages.items():
                if selector_usages.__contains__(key):
                    selector_usages[key].append(value)
                else:
                    selector_usages[key] = value
    return selector_usages

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

def find_component_paths_in_routes(dir):
    result = {}
    files = os.listdir(dir)
    for file in files:
        file_path = os.path.join(dir, file)
        path_regex = r'path:\s*[\'"](.*)[\'"],?'
        component_regex = r'component:\s*([A-Za-z]*),?'

        if os.path.isfile(file_path) and file.endswith('-routing.module.ts'):
            with open(file_path) as f:
                list_started = False
                path_dict = {}
                for line in f:
                    trimmed_line = line.strip()
                    if list_started:
                        if trimmed_line.startswith('{'):
                            path_dict = {}
                        elif 'path:' in trimmed_line:
                            match = re.search(path_regex, trimmed_line)
                            if match:
                                path_dict['path'] = match.group(1)
                            else:
                                print('RIDI')
                        elif 'component:' in trimmed_line:
                            match = re.search(component_regex, trimmed_line)
                            if match:
                                path_dict['component'] = match.group(1)
                            else:
                                print('RIDI')
                        elif trimmed_line.startswith('}'):
                            if path_dict.__contains__('component'):
                                result[path_dict['component']] = path_dict['path']
                        elif trimmed_line.startswith(']'):
                            break
                    elif trimmed_line.endswith('['):
                        list_started = True
        elif os.path.isdir(file_path):
            paths = find_component_paths_in_routes(file_path)
            result.update(paths)
    return result

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

            selector = find_component_selector(file_path)
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
                if method_details.__contains__(selector):
                    method_details[selector]['method'].append(method)
                    method_details[selector]['url'].append(url)
                else:
                    method_details[selector] = {'file': file_path, 'url': [].append(url), 'method': [].append(method)}

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
paths_dict = find_component_paths_in_routes(angular_dir)

not_found_components = []

# for method, details in method_details.items():
#     end_components = track_method_usages(method_details, method_usages, method, angular_dir)
#     component_results = []
# 
#     for component_file, status in end_components:
#         if status == "PAGE":
#             component_name_match = re.search(r'export\s+class\s+(\w+)\s+', open(component_file).read())
#             component_name = component_name_match.group(1) if component_name_match else None
#             if component_name and paths_dict.__contains__(component_name):
#                 route_path = paths_dict[component_name]
#                 component_results.append((component_name, route_path))
#             else:
#                 not_found_components.append(component_name)
# 
#     final_results[method] = {
#         'url': details['url'],
#         'components': component_results
#     }
end_components = track_method_usages_v2(method_details, angular_dir)

# Display results
if final_results:
    for method, details in final_results.items():
        print("URL: {}".format(details['url']))
        print("USAGE:")
        for component, path in details['components']:
            print("  component: {}".format(component))
            print("  path: \"{}\"".format(path))
        print('\n--------------------------------------------------------------\n')
else:
    print('No apiService calls found.')

print('----------------------------------')
print('Components That Are Not Found')
print('----------------------------------')
for component in not_found_components:
    print(component)



```
