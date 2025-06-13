#!/pkg/qct/software/python/sles12/3.13.1/bin/python3 -E

import os
import re
import sys
import argparse
import gzip

def print_help():
    help_text = """
 Usage:
  python final.py --path {parse_include_env_plus, expand_parse_commandline, both_cli_parse, expanding_allfiles_pre_env_txt}
                  --input_path 'INPUT_PATH'
                  --output 'OUTPUT'
                  [--log 'LOG']
                  [--env_file 'ENV_FILE']

Options:
  --path         Script to run (choose one of the following):
                   - parse_include_env_plus
                   - expand_parse_commandline
                   - both_cli_parse
                   - expanding_allfiles_pre_env_txt
  --input_path   Input file (.sl, .dl, or Makefile)
  --output       Output file
  --log          Log file with command line (used by some scripts)
  --env_file     Environment file (required for expanding_allfiles_pre_env_txt)

Examples:
  1. For `parse_include_env_plus`:Expands includes and environment variables
     Script --path=parse_include_env_plus --input_path=my.sl --output=expanded.txt

  2. For `expand_parse_commandline`:Parses of expands command-line arguments
     Script --path=expand_parse_commandline --input_path=my.dl --log=my.log --output=exp.txt

  3. For `both_cli_parse`:Combines include and command-line arguments parsing
     Script --path=both_cli_parse --input_path=my.sl --log=my.log --output=all.txt

  4. For `expanding_allfiles_pre_env_txt`:Expands files using environment file
     Script --path=expanding_allfiles_pre_env_txt --input_path=my.dl --output=all_exp.txt --env_file=pre_qvm_regress_report_env.txt
""".strip()
    print(help_text)

def file_is_empty(path):
    try:
        return os.path.exists(path) and os.path.getsize(path) == 0
    except Exception:
        return False

def expand_env(line):
    def repl(match):
        var = match.group(1) or match.group(2)
        return os.environ.get(var, match.group(0))
    return re.sub(r'\$\{(\w+)\}|\$(\w+)', repl, line)

def expand_env_custom(line, fallback=None):
    def repl_brace(m):
        var = m.group(1)
        if fallback:
            return os.environ.get(var, fallback(var))
        return os.environ.get(var, var)
    line = re.sub(r'\$\{(\w+)\}', repl_brace, line)
    def repl_plain(m):
        var = m.group(1)
        if fallback:
            return os.environ.get(var, fallback(var))
        return os.environ.get(var, var)
    line = re.sub(r'\$(\w+)', repl_plain, line)
    return line

def expand_path(path):
    return os.path.abspath(os.path.expanduser(expand_env(path.strip())))

def expand_path_with_env_variable(path, fallback=None):
    expanded = expand_env_custom(path.strip(), fallback)
    return os.path.abspath(expanded)

def safe_read_file(path):
    path = expand_path(path)
    if not os.path.exists(path):
        sys.stderr.write("File not found - {}\n".format(path))
        return []
    if os.path.isdir(path):
        sys.stderr.write("Debug: Path is a directory - {}\n".format(path))
        return []
    if file_is_empty(path):
        sys.stderr.write("Hey, {} is provided empty file. Check and update correct one.\n".format(path))
        return []
    ext = os.path.splitext(path)[1].lower()
    try:
        if ext == ".gz":
            with gzip.open(path, "rt", encoding="utf-8") as f:
                return f.readlines()
        elif ext in [".bin", ".dat"]:
            with open(path, "rb") as f:
                content = f.read()
                try:
                    return content.decode("utf-8").splitlines(keepends=True)
                except UnicodeDecodeError:
                    return content.decode("latin-1").splitlines(keepends=True)
        else:
            try:
                with open(path, "r", encoding="utf-8") as f:
                    return f.readlines()
            except UnicodeDecodeError:
                with open(path, "r", encoding="latin-1") as f:
                    return f.readlines()
    except PermissionError:
        sys.stderr.write("Permission denied: {}\n".format(path))
        return []
    except Exception as e:
        sys.stderr.write("Unknown error reading {}: {}\n".format(path, str(e)))
        return []

def safe_read_entire_file(path):
    path = expand_path(path)
    if not os.path.exists(path):
        sys.stderr.write("File not found - {}\n".format(path))
        return ""
    ext = os.path.splitext(path)[1].lower()
    try:
        if ext == ".gz":
            with gzip.open(path, "rt", encoding="utf-8") as f:
                return f.read()
        else:
            try:
                with open(path, "r", encoding="utf-8") as f:
                    return f.read()
            except UnicodeDecodeError:
                with open(path, "r", encoding="latin-1") as f:
                    return f.read()
    except Exception as e:
        sys.stderr.write("Unknown error reading {}: {}\n".format(path, str(e)))
        return ""

def process_includes(path, processed=None, fallback=None):
    if processed is None:
        processed = set()
    expanded = expand_path_with_env_variable(path, fallback)
    if expanded in processed:
        return []
    processed.add(expanded)
    lines = safe_read_file(expanded)
    result = []
    include_re1 = re.compile(r'^\s*`?include\s+(["\']?)(.+?)\1\s*$')
    include_re2 = re.compile(r'include\s+["\'](.+?)["\']')
    for line in lines:
        m1 = include_re1.match(line)
        m2 = include_re2.search(line)
        if m1:
            inc = expand_path_with_env_variable(m1.group(2), fallback)
            result.append("# INCLUDE PATH: {}\n".format(inc))
            result.extend(process_includes(inc, processed, fallback))
        elif m2:
            inc = expand_path_with_env_variable(m2.group(1), fallback)
            result.append("# INCLUDE PATH: {}\n".format(inc))
            result.extend(process_includes(inc, processed, fallback))
        else:
            result.append(line)
    return result

def extract_plus_groups(lines):
    group_defs = {}
    pattern = re.compile(r'^(\w+)\s*\{')
    i = 0
    while i < len(lines):
        line = lines[i]
        match = pattern.match(line.strip())
        if match:
            group_name = match.group(1)
            block_lines = []
            brace_count = 0
            brace_count += line.count('{') - line.count('}')
            block_lines.append(line[line.index('{')+1:].rstrip())
            i += 1
            while i < len(lines) and brace_count > 0:
                l = lines[i]
                brace_count += l.count('{') - l.count('}')
                block_lines.append(l)
                i += 1
            if block_lines and block_lines[-1].strip().endswith('}'):
                block_lines[-1] = block_lines[-1].rstrip().rstrip('}')
            content = ''.join(block_lines).strip('\n')
            group_defs[group_name] = content
        else:
            i += 1
    return group_defs

def recursive_expand_group_content(content, group_defs, expansion_stack=None, fallback=None):
    if expansion_stack is None:
        expansion_stack = set()
    lines = content.splitlines(keepends=True)
    plus_pattern = re.compile(r'^\s*\+(\w+)\s*$')
    result = []
    for line in lines:
        m = plus_pattern.match(line.strip())
        if m:
            groupname = m.group(1)
            if groupname in expansion_stack:
                result.append(f"# [WARNING: Recursive group reference: {groupname}]\n")
                continue
            nested_content = group_defs.get(groupname)
            if nested_content is not None:
                result.append(f"# Expanding element {groupname}\n")
                expanded = recursive_expand_group_content(
                    nested_content, group_defs, expansion_stack | {groupname}, fallback
                )
                for l in expanded:
                    result.append("   " + l.lstrip())
            else:
                result.append(line)
        else:
            result.append(line)
    return result

def expand_plus_groups(lines, group_defs, fallback=None):
    result = []
    plus_pattern = re.compile(r'^\s*\+(\w+)\s*$')
    for line in lines:
        m = plus_pattern.match(line.strip())
        if m:
            groupname = m.group(1)
            content = group_defs.get(groupname)
            if content is not None:
                result.append(f"# Expanding element {groupname}\n")
                expanded = recursive_expand_group_content(content, group_defs, {groupname}, fallback)
                for l in expanded:
                    result.append("   " + l.lstrip())
            else:
                result.append(line)
        else:
            result.append(line)
    return result

def parse_include_env_plus(input_path, output_path):
    basename = os.path.basename(input_path)
    fallback = lambda var: basename
    processed_lines = process_includes(input_path, fallback=fallback)
    group_defs = extract_plus_groups(processed_lines)
    expanded_lines = expand_plus_groups(processed_lines, group_defs, fallback=fallback)
    final_lines = [expand_env_custom(line, fallback=fallback) for line in expanded_lines]
    if not output_path.lower().endswith('.txt'):
        output_path += '.txt'
    with open(output_path, 'w', encoding='utf-8') as f:
        for line in final_lines:
            f.write(line if line.endswith('\n') else line+'\n')
    print(f"Expanded includes, +groups, and env vars written to {output_path}")

def expanding_allfiles_pre_env_txt(input_path, output_path, env_file):
    # Load environment variables from env_file
    if not env_file:
        print("Error: --env_file is required for expanding_allfiles_pre_env_txt")
        sys.exit(1)
    env_file_path = env_file
    if env_file_path.startswith("--log="):
        env_file_path = env_file_path.replace("--log=", "")
    env_file_path = expand_path(env_file_path)
    if not os.path.exists(env_file_path):
        print(f"Error: Environment file '{env_file_path}' not found.")
        sys.exit(1)
    with open(env_file_path, "r", encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if not line or line.startswith("#"):
                continue
            if "=" not in line:
                continue
            key, val = line.split('=', 1)
            val = val.strip()
            if (val.startswith('"') and val.endswith('"')) or (val.startswith("'") and val.endswith("'")):
                val = val[1:-1]
            os.environ[key.strip()] = val
    # Use the main recursive expansion logic
    basename = os.path.basename(input_path)
    fallback = lambda var: basename
    processed_lines = process_includes(input_path, fallback=fallback)
    group_defs = extract_plus_groups(processed_lines)
    expanded_lines = expand_plus_groups(processed_lines, group_defs, fallback=fallback)
    final_lines = [expand_env_custom(line, fallback=fallback) for line in expanded_lines]
    if not output_path.lower().endswith('.txt'):
        output_path += '.txt'
    with open(output_path, 'w', encoding='utf-8') as f:
        for line in final_lines:
            f.write(line if line.endswith('\n') else line+'\n')
    print(f"Expanded (with pre-env) includes, +groups, and env vars written to {output_path}")

# script2, script3, helpers unchanged from your prior version...

def extract_plus_groups_with_linenum(lines):
    group_defs = {}
    pattern = re.compile(r'^(\w+)\s*\{')
    i = 0
    while i < len(lines):
        line = lines[i]
        match = pattern.match(line.strip())
        if match:
            group_name = match.group(1)
            block_lines = []
            brace_count = 0
            brace_count += line.count('{') - line.count('}')
            block_lines.append(line[line.index('{')+1:].rstrip())
            start_lineno = i + 1  # 1-based
            i += 1
            while i < len(lines) and brace_count > 0:
                l = lines[i]
                brace_count += l.count('{') - l.count('}')
                block_lines.append(l)
                i += 1
            if block_lines and block_lines[-1].strip().endswith('}'):
                block_lines[-1] = block_lines[-1].rstrip().rstrip('}')
            content = ''.join(block_lines).strip('\n')
            group_defs[group_name] = (content, start_lineno)
        else:
            i += 1
    return group_defs

def extract_commandline_plus_groups(log_file):
    log_file = expand_path(log_file)
    try:
        with open(log_file, 'r', encoding='utf-8') as infile:
            for line in infile:
                if 'Commandline:' in line:
                    command_line = line.split('Commandline:', 1)[1].strip()
                    group_options = re.findall(r'\+(\w+)', command_line)
                    return set(group_options)
    except Exception:
        try:
            with open(log_file, 'r', encoding='latin1') as infile:
                for line in infile:
                    if 'Commandline:' in line:
                        command_line = line.split('Commandline:', 1)[1].strip()
                        group_options = re.findall(r'\+(\w+)', command_line)
                        return set(group_options)
        except Exception:
            pass
    return set()

def extract_regress_blocks(content):
    pattern = re.compile(r'(REGRESS_OPTIONS\s*\{[^}]*\})', re.DOTALL)
    return pattern.findall(content)

FORBIDDEN_OPTIONS = ['-build_memory', '-run_memory', '-build_bsub_args', '-run_bsub_args']
def should_skip_line(line):
    return any(opt in line for opt in FORBIDDEN_OPTIONS)

def collect_all_groups(main_file):
    all_group_defs = {}
    processed_files = set()
    def _collect(path):
        abs_path = expand_path(path)
        if abs_path in processed_files:
            return
        processed_files.add(abs_path)
        lines = safe_read_file(abs_path)
        group_defs = extract_plus_groups_with_linenum(lines)
        for g, v in group_defs.items():
            if g not in all_group_defs:
                all_group_defs[g] = v
        include_pattern1 = re.compile(r'^\s*`?include\s+(["\']?)(.+?)\1\s*$')
        include_pattern2 = re.compile(r'include\s+["\'](.+?)["\']')
        for line in lines:
            match1 = include_pattern1.match(line)
            match2 = include_pattern2.search(line)
            if match1:
                include_file = expand_path(match1.group(2))
                _collect(include_file)
            elif match2:
                include_file = expand_path(match2.group(1))
                _collect(include_file)
    _collect(main_file)
    return all_group_defs

def expand_group_content_with_own_lineno(group, group_defs, group_lookup, expanded_groups=None, file_path=None):
    if expanded_groups is None:
        expanded_groups = set()
    content, start_lineno = group_defs[group]
    lines = content.splitlines(keepends=True)
    plus_pattern = re.compile(r'^\s*\+(\w+)\s*$')
    result = []
    for idx, line in enumerate(lines):
        m = plus_pattern.match(line.strip())
        if m:
            groupname = m.group(1)
            if groupname in expanded_groups:
                result.append(f"# [WARNING: Recursive group reference: {groupname}]\n")
                continue
            group_defs2, file_path2 = group_lookup(groupname)
            if group_defs2 is not None:
                result.append(f"# Expanding_element {groupname} from_the_path :-- {file_path2}\n")
                result.append("---------------------------------------------------------------------------------------------------\n")
                expanded = expand_group_content_with_own_lineno(
                    groupname, group_defs2, group_lookup,
                    expanded_groups | {groupname},
                    file_path2
                )
                for l in expanded:
                    result.append("   " + l.lstrip())
            else:
                result.append(line)
        else:
            result.append(line)
    return result

def expand_regress_block(block_content, group_lookup, expanded_groups, file_path):
    plus_pattern = re.compile(r'^\s*\+(\w+)\s*$')
    lines = block_content.splitlines()
    result = []
    for line in lines:
        if should_skip_line(line):
            continue
        m = plus_pattern.match(line.strip())
        if m:
            groupname = m.group(1)
            group_defs2, file_path2 = group_lookup(groupname)
            if group_defs2 is not None:
                result.append(f"# Expanding_element {groupname} from_the_path :-- {file_path2}\n")
                result.append("---------------------------------------------------------------------------------------------------\n")
                expanded = expand_group_content_with_own_lineno(
                    groupname, group_defs2, group_lookup,
                    expanded_groups | {groupname} if expanded_groups else {groupname},
                    file_path2
                )
                for l in expanded:
                    result.append("   " + l.lstrip())
            else:
                result.append("   " + line.rstrip() + "\n")
        else:
            result.append("   " + line.rstrip() + "\n")
    return result

def script2(dl_file, log_file, output_file):
    allowed = (
        dl_file.endswith('.sl') or
        dl_file.endswith('.dl') or
        os.path.basename(dl_file) == 'Makefile'
    )
    if not allowed or not os.path.isfile(expand_path(dl_file)):
        print_help()
        return

    commandline_groups = extract_commandline_plus_groups(log_file)
    all_group_defs = {}
    all_group_locations = {}
    already_processed_files = set()

    def collect_group_defs(file_path):
        abs_path = expand_path(file_path)
        if abs_path in already_processed_files:
            return
        already_processed_files.add(abs_path)
        lines = safe_read_file(abs_path)
        group_defs = extract_plus_groups_with_linenum(lines)
        for group in group_defs:
            all_group_locations[group] = (group_defs, abs_path)
        include_pattern1 = re.compile(r'^\s*`?include\s+(["\']?)(.+?)\1\s*$')
        include_pattern2 = re.compile(r'include\s+["\'](.+?)["\']')
        for line in lines:
            match1 = include_pattern1.match(line)
            match2 = include_pattern2.search(line)
            if match1:
                include_file = expand_path(match1.group(2))
                collect_group_defs(include_file)
            elif match2:
                include_file = expand_path(match2.group(1))
                collect_group_defs(include_file)
    collect_group_defs(dl_file)

    def group_lookup(groupname):
        return all_group_locations.get(groupname, (None, None))

    output_lines = []
    full_dl_path = expand_path(dl_file)
    output_lines.append("# Input file: {}\n\n".format(full_dl_path))

    ext = os.path.splitext(dl_file)[1]
    base = os.path.basename(dl_file)
    allowed = ext in ('.dl', '.sl') or base == 'Makefile'
    if allowed:
        content = safe_read_entire_file(dl_file)
        regress_blocks = extract_regress_blocks(content)
        if regress_blocks:
            output_lines.append("# REGRESS_OPTIONS blocks found in input file:\n")
            for block in regress_blocks:
                block_inside = re.search(r'REGRESS_OPTIONS\s*\{([^}]*)\}', block, re.DOTALL)
                if block_inside:
                    block_content = block_inside.group(1)
                    output_lines.extend(expand_regress_block(block_content, group_lookup, set(), dl_file))
                else:
                    output_lines.append(block.strip() + '\n' + '-'*60 + '\n')
            output_lines.append("\n")
        else:
            output_lines.append("# No REGRESS_OPTIONS found in input file.\n\n")

    for group in sorted(commandline_groups):
        group_defs, file_path = all_group_locations.get(group, (None, None))
        if group_defs and file_path:
            content, start_lineno = group_defs[group]
            output_lines.append("\n# Expanding_element {} from_the_path :-- {}\n".format(group, file_path))
            output_lines.append("---------------------------------------------------------------------------------------------------\n")
            expanded = expand_group_content_with_own_lineno(
                group, group_defs, group_lookup,
                expanded_groups=None,
                file_path=file_path
            )
            output_lines.extend(expanded)
        else:
            output_lines.append("# [WARNING: +{} not found in any file]\n".format(group))

    try:
        with open(output_file, 'w') as f:
            f.writelines(output_lines)
        print("Expanded groups written to {}".format(output_file))
    except Exception as e:
        sys.stderr.write("Could not write to {}: {}\n".format(output_file, str(e)))
        sys.exit(2)

def script3(input_file, log_file, output_file):
    allowed = (
        input_file.endswith('.sl') or
        input_file.endswith('.dl') or
        os.path.basename(input_file) == 'Makefile'
    )
    if not allowed or not os.path.isfile(expand_path(input_file)):
        print_help()
        return
    all_groups = collect_all_groups(input_file)
    lines = process_includes(input_file)
    expanded_env_lines = [expand_env(line) for line in lines]
    plus_pat = re.compile(r'^\s*\+(\w+)\s*$')
    expanded_plus_groups_lines = []
    def recursive_expand_group_content_both(content, all_groups, stack=None):
        if stack is None:
            stack = set()
        lines = content.splitlines(keepends=True)
        plus_pat = re.compile(r'^\s*\+(\w+)\s*$')
        result = []
        for line in lines:
            m = plus_pat.match(line.strip())
            if m:
                gname = m.group(1)
                if gname in stack:
                    result.append("# [WARNING: Recursive group reference: {}]\n".format(gname))
                    continue
                nested_content, _ = all_groups.get(gname, (None, None))
                if nested_content is not None:
                    result.append("# Expanding element {}\n".format(gname))
                    expanded = recursive_expand_group_content_both(nested_content, all_groups, stack | {gname})
                    for l in expanded:
                        result.append("   " + l.lstrip())
                else:
                    result.append(line)
            else:
                result.append(line)
        joined = ''.join(result)
        regress_blocks = extract_regress_blocks(joined)
        if regress_blocks:
            expanded = []
            for block_content in regress_blocks:
                expanded.append("REGRESS_OPTIONS {\n")
                expanded.extend(expand_regress_block_both(block_content, all_groups, recursive_expand_group_content_both))
                expanded.append("}\n" + "-"*60 + "\n")
            return expanded
        return result
    def expand_regress_block_both(content, all_groups, recursive_func):
        plus_pat = re.compile(r'^\s*\+(\w+)\s*$')
        lines = content.splitlines()
        result = []
        for line in lines:
            if should_skip_line(line):
                continue
            m = plus_pat.match(line.strip())
            if m:
                gname = m.group(1)
                nested_content, _ = all_groups.get(gname, (None, None))
                if nested_content is not None:
                    result.append("   # Expanding element {}\n".format(gname))
                    expanded = recursive_func(nested_content, all_groups, {gname})
                    for l in expanded:
                        result.append("   " + l.lstrip())
                else:
                    result.append("   " + line.rstrip() + "\n")
            else:
                result.append("   " + line.rstrip() + "\n")
        return result
    for line in expanded_env_lines:
        m = plus_pat.match(line.strip())
        if m:
            gname = m.group(1)
            content, _ = all_groups.get(gname, (None, None))
            if content is not None:
                expanded_plus_groups_lines.append("# Expanding element {}\n".format(gname))
                expanded = recursive_expand_group_content_both(content, all_groups, {gname})
                for l in expanded:
                    expanded_plus_groups_lines.append("   " + l.lstrip())
            else:
                expanded_plus_groups_lines.append(line)
        else:
            expanded_plus_groups_lines.append(line)
    content = ''.join(expanded_env_lines)
    regress_blocks = extract_regress_blocks(content)
    expanded_regress_section = []
    if regress_blocks:
        expanded_regress_section.append("# REGRESS_OPTIONS blocks expanded (with +groups):\n")
        for block_content in regress_blocks:
            expanded_regress_section.append("REGRESS_OPTIONS {\n")
            expanded_regress_section.extend(expand_regress_block_both(block_content, all_groups, recursive_expand_group_content_both))
            expanded_regress_section.append("}\n" + "-"*60 + "\n")
    else:
        expanded_regress_section.append("# No REGRESS_OPTIONS found in input file or includes.\n")
    cmd_groups = extract_commandline_plus_groups(log_file)
    expanded_cmd_groups_section = []
    if cmd_groups:
        expanded_cmd_groups_section.append("\n\n# Expansion of +groups from Commandline in log file (regress_options already above):\n")
        for group in sorted(cmd_groups):
            content, _ = all_groups.get(group, (None, None))
            if content:
                expanded_cmd_groups_section.append("\n# Expanding_element {}\n".format(group))
                expanded = recursive_expand_group_content_both(content, all_groups, {group})
                for l in expanded:
                    expanded_cmd_groups_section.append(l if l.endswith('\n') else l+'\n')
            else:
                expanded_cmd_groups_section.append("# [WARNING: +{} not found in any file]\n".format(group))
    else:
        expanded_cmd_groups_section.append("\n# No +groups found in Commandline in log file.\n")
    with open(output_file, 'w', encoding="utf-8") as f:
        f.write("# Full file expansion (includes, env vars, +groups):\n\n")
        for line in expanded_plus_groups_lines:
            f.write(line if line.endswith('\n') else line+'\n')
        for line in expanded_regress_section:
            f.write(line if line.endswith('\n') else line+'\n')
        for line in expanded_cmd_groups_section:
            f.write(line if line.endswith('\n') else line+'\n')
    print("Expanded (both_cli_parse style) output written to {}".format(output_file))

def main():
    parser = argparse.ArgumentParser(
        description="Expand include directives, env vars, +groups, and command-line arguments for SL/DL/Makefile flows",
        add_help=False
    )
    parser.add_argument('--path', required=False, choices=[
        'parse_include_env_plus',
        'expand_parse_commandline',
        'both_cli_parse',
        'expanding_allfiles_pre_env_txt'
    ], help='Script to run')
    parser.add_argument('--input_path', required=False, help='Input .sl, .dl, or Makefile')
    parser.add_argument('--output', required=False, help='Output file')
    parser.add_argument('--log', help='Log file with Commandline (for some scripts)')
    parser.add_argument('--env_file', help='Environment file (for expanding_allfiles_pre_env_txt)')
    parser.add_argument('-h', '--help', action='store_true', help='Show help message and exit')
    args = parser.parse_args()
    if args.help or (len(sys.argv) == 2 and sys.argv[1] in ("--help", "-h")):
        print_help()
        sys.exit(0)
    if args.path == "expand_parse_commandline":
        if not args.input_path or not args.log or not args.output:
            print_help()
            sys.exit(1)
        script2(args.input_path, args.log, args.output)
    elif args.path == "both_cli_parse":
        if not args.input_path or not args.log or not args.output:
            print_help()
            sys.exit(1)
        script3(args.input_path, args.log, args.output)
    elif args.path == "parse_include_env_plus":
        if not args.input_path or not args.output:
            print_help()
            sys.exit(1)
        parse_include_env_plus(args.input_path, args.output)
    elif args.path == "expanding_allfiles_pre_env_txt":
        if not args.input_path or not args.output or not args.env_file:
            print_help()
            sys.exit(1)
        expanding_allfiles_pre_env_txt(args.input_path, args.output, args.env_file)
    else:
        print_help()
        sys.exit(1)

if __name__ == "__main__":
    main()
