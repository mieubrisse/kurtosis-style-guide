**WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING**

Shell is a terrible language for programming - it offers very little guardrails, and it's incredibly easy to shoot yourself in the foot - often without even realizing what's happening.
This isn't to say that you shouldn't write programs in shell - it's the most portable language around. However, you must be INCREDIBLY careful: familiarize yourself with this guide, code carefully and defensively, and don't be afraid to ask for second eyes on the script!

**WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING**

### All shell scripts must start from [the Bash template](https://github.com/kurtosis-tech/checklists-and-templates/blob/master/generic-bash-script.sh), which enforces strict mode
* Rationale: 
    * Bash strict mode is **essential** to writing safe Bash code
        * `set -e` means "exit the script with an error if any command fails" (rather than continuing on as if nothing happened, which is the shell defeault)
        * `set -o pipefail` means "exit the script with an error, even if the command that failed was inside a pipe"
            * E.g. `echo "foo" | grep "bar"`, which would normally pass due to the `echo` succeeding even though the `grep` failed
        * `set -u` means "exit the script with an error if an undeclared variable is used"
            * E.g. `set -u; echo "${my_variable}"` will fail while `set -u; my_variable="some value"; echo "${my_variable}"` will succeed
    * Standardizes our Bash scripts


### Any time you use `rm -rf` with a variable, you MUST check that the variable isn't accidentally empty**
* Rationale: if you don't, you can accidentally call `rm -rf`!!
    * E.g. `rm -rf "${accidentally_empty_variable}/"` translates to `rm -rf /`
* Much better is the following (but you should still normally avoid any `rm -rf` in your Bash scripts!):

    ```
    if [ "${accidentally_empty_variable}/" = "/" ]; then
        echo "ERROR: Would have ran 'rm -rf /'!!" >&2
        exit 1
    fi
    rm -rf "${accidentally_empty_variable}/"
    ```

### Use the single-brace test operator `[` over `[[`
* Rationale: `[` is portable across all shells, while `[[` is Bash-specific

### Getting a script's directory should be done via `script_dirpath="$(cd "$(dirname "${0}")" && pwd)"`
* Rationale: this seems to be the best way 

### Use `$()` for getting process output, rather than backticks
* Rationale: Backticks aren't nestable, while `$()` is, e.g.:

    ```bash
    # Impossible to do with backticks
    myvar="$(echo "$(grep 'thevalue' some-file.txt)")"
    ```

### Constant variable names should be uppercased, while non-constants should be lowercased
* Rationale: makes it visually clear which values should be changed, and which shouldn't
* E.g. `CONST="foo"` or `non_constant="mutable value: ${CONST}"`

### Bash variables should always be wrapped in double quotes and referenced via `${}` (e.g. `echo "${some_variable}"`), UNLESS you explicitly want word-splitting (very rare)
* The double quotes protects you from accidental [word-splitting](https://www.gnu.org/software/bash/manual/html_node/Word-Splitting.html)
* The `${}` prevents you from tricky variable name reference errors
    * E.g. you'd expect `logfile_name="log"; echo "Some logline" > $logfile_name_$(date +%F_%H-%m-%s).log` to output a file named `log_2021-10-29_16-45-10.log`, but will actually output a file named `2021-10-29_16-45-10.log` because Bash thinks it should expand the variable `$logfile_name_` (note: trailing underscore) which doesn't exist and will be empty
* Intentional word-splitting usually only happens in a `for` loop
    * E.g. `values="value1 value2 value3"; for value in ${values}; do echo "Value: ${value}"; done` - notice the lack of `"` around `${values}`
