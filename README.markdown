FitNesse Plugin for Jenkins
==============================

See [Jenkins wiki](http://wiki.jenkins-ci.org/display/JENKINS/FitNesse+Plugin) for details.


Świetny kontekst! W takim razie potrzebujesz frameworku do **testów integracyjnych klienta z serwisem**, który pozwoli **testerom pisać scenariusze** bez kodowania w C++. Oto optymalne rozwiązanie:

## 🎯 **Architektura: Python + C++ Binding + YAML/JSON scenariusze**

### **Dlaczego to idealne rozwiązanie:**
- **Testerzy piszą scenariusze w YAML/JSON** (bez kodu)
- **Biblioteka C++ jest testowana bezpośrednio**
- **Możliwość testowania przeciwko różnym środowiskom API**
- **Raporty w formacie dla ludzi**

## 🏗️ **Struktura projektu**

```
project/
├── src/                    # Kod C++ biblioteki
├── tests/
│   ├── cpp/               # Testy jednostkowe C++ (Google Test)
│   ├── integration/       # Testy integracyjne
│   │   ├── scenarios/     # Scenariusze YAML/JSON
│   │   ├── runners/       # Python runner dla scenariuszy
│   │   └── bindings/      # Python binding do biblioteki C++
│   └── fixtures/          # Dane testowe, mock serwery
```

## 🔧 **Implementacja krok po kroku**

### **1. Python Binding do biblioteki C++ (przy użyciu pybind11)**

```cpp
// tests/integration/bindings/python_binding.cpp
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>
#include "your_library_client.h"

namespace py = pybind11;

PYBIND11_MODULE(client_lib, m) {
    py::class_<LibraryClient>(m, "LibraryClient")
        .def(py::init<const std::string&>())  // constructor z URL
        .def("connect", &LibraryClient::connect)
        .def("authenticate", &LibraryClient::authenticate)
        .def("send_data", &LibraryClient::sendData)
        .def("receive_data", &LibraryClient::receiveData)
        .def("disconnect", &LibraryClient::disconnect);
    
    py::class_<AuthResult>(m, "AuthResult")
        .def_readonly("success", &AuthResult::success)
        .def_readonly("session_id", &AuthResult::sessionId)
        .def_readonly("error_message", &AuthResult::errorMessage);
}
```

### **2. System scenariuszy YAML dla testerów**

```yaml
# tests/integration/scenarios/authentication_flow.yaml
name: "Authentication Flow Test"
description: "Test complete authentication scenario"
api_url: "https://api-staging.yourservice.com"
timeout: 30

steps:
  - name: "Connect to service"
    action: "connect"
    parameters:
      endpoint: "/api/v1/connect"
    expected:
      status: "connected"
      timeout: 5000

  - name: "Successful authentication"
    action: "authenticate" 
    parameters:
      username: "test_user"
      password: "test_pass_123"
      role: "admin"
    expected:
      success: true
      session_id: "not_empty"
      permissions: ["read", "write", "delete"]

  - name: "Failed authentication - wrong password"
    action: "authenticate"
    parameters:
      username: "test_user"
      password: "wrong_password"
    expected:
      success: false
      error_code: "AUTH_FAILED"
      session_id: "empty"

  - name: "Cleanup - disconnect"
    action: "disconnect"
    expected:
      status: "disconnected"
```

```yaml
# tests/integration/scenarios/data_synchronization.yaml
name: "Data Sync Scenario"
description: "Test data synchronization with service"

steps:
  - name: "Setup connection"
    action: "connect"
    parameters:
      endpoint: "/api/v1/connect"

  - name: "Send small data batch"
    action: "send_data"
    parameters:
      data_type: "user_profile"
      payload:
        user_id: 12345
        name: "John Doe"
        preferences: {"theme": "dark", "language": "en"}
      batch_size: 1
    expected:
      status: "accepted"
      message_id: "not_empty"

  - name: "Send large data batch"
    action: "send_data"
    parameters:
      data_type: "bulk_operations"
      payload_file: "fixtures/large_dataset.json"
      batch_size: 1000
    expected:
      status: "processing"
      estimated_time: "> 60"

  - name: "Verify data received by service"
    action: "verify_server_state"
    parameters:
      verification_url: "https://api-staging.yourservice.com/verify/12345"
      expected_state: "processed"
    expected:
      status: "verified"
      data_consistency: true
```

### **3. Python Runner dla scenariuszy**

```python
# tests/integration/runners/scenario_runner.py
import yaml
import json
import time
import requests
from client_lib import LibraryClient  # Nasz binding C++

class ScenarioRunner:
    def __init__(self, base_url, report_dir="reports"):
        self.client = LibraryClient(base_url)
        self.report_dir = report_dir
        self.results = []
    
    def run_scenario(self, scenario_file):
        """Główna metoda uruchamiająca scenariusz"""
        with open(scenario_file, 'r') as f:
            scenario = yaml.safe_load(f)
        
        print(f"🚀 Running scenario: {scenario['name']}")
        print(f"📝 Description: {scenario['description']}")
        
        scenario_results = {
            'scenario_name': scenario['name'],
            'start_time': time.time(),
            'steps': []
        }
        
        for step in scenario['steps']:
            step_result = self._execute_step(step)
            scenario_results['steps'].append(step_result)
            
            if not step_result['success']:
                print(f"❌ Step failed: {step['name']}")
                break
            else:
                print(f"✅ Step passed: {step['name']}")
        
        scenario_results['end_time'] = time.time()
        scenario_results['success'] = all(step['success'] for step in scenario_results['steps'])
        
        self._generate_report(scenario_results, scenario_file)
        return scenario_results
    
    def _execute_step(self, step):
        """Wykonuje pojedynczy krok scenariusza"""
        step_result = {
            'name': step['name'],
            'action': step['action'],
            'success': False,
            'timestamp': time.time(),
            'details': {}
        }
        
        try:
            if step['action'] == 'connect':
                result = self.client.connect(step['parameters']['endpoint'])
                step_result['success'] = self._check_expectations(result, step['expected'])
                
            elif step['action'] == 'authenticate':
                result = self.client.authenticate(
                    step['parameters']['username'],
                    step['parameters']['password']
                )
                step_result['success'] = self._check_expectations(result, step['expected'])
                
            elif step['action'] == 'send_data':
                # Obsługa payload z pliku lub bezpośrednio
                if 'payload_file' in step['parameters']:
                    with open(step['parameters']['payload_file'], 'r') as f:
                        payload = json.load(f)
                else:
                    payload = step['parameters']['payload']
                
                result = self.client.send_data(
                    step['parameters']['data_type'],
                    payload
                )
                step_result['success'] = self._check_expectations(result, step['expected'])
                
            elif step['action'] == 'verify_server_state':
                # Weryfikacja stanu po stronie serwera przez HTTP
                response = requests.get(step['parameters']['verification_url'])
                step_result['success'] = (response.status_code == 200 and 
                                        response.json()['state'] == step['expected']['status'])
            
            step_result['details'] = {'result': result.__dict__ if hasattr(result, '__dict__') else result}
            
        except Exception as e:
            step_result['error'] = str(e)
            step_result['details'] = {'exception': type(e).__name__}
        
        return step_result
    
    def _check_expectations(self, actual_result, expected):
        """Sprawdza czy wynik spełnia oczekiwania"""
        for key, expected_value in expected.items():
            actual_value = getattr(actual_result, key, None)
            
            if expected_value == "not_empty":
                if not actual_value:
                    return False
            elif expected_value == "empty":
                if actual_value:
                    return False
            elif str(expected_value).startswith(">"):
                # Obsługa warunków jak "> 60"
                threshold = int(expected_value[1:])
                if not (actual_value > threshold):
                    return False
            elif actual_value != expected_value:
                return False
                
        return True
    
    def _generate_report(self, results, scenario_file):
        """Generuje raport w formacie HTML/JSON"""
        report_file = f"{self.report_dir}/{scenario_file.stem}_{int(time.time())}.json"
        
        with open(report_file, 'w') as f:
            json.dump(results, f, indent=2)
        
        # Generuj też raport HTML dla testerów
        self._generate_html_report(results, report_file.replace('.json', '.html'))
        
        print(f"📊 Report generated: {report_file}")
```

### **4. Interfejs CLI dla testerów**

```python
# tests/integration/runners/cli.py
import click
from pathlib import Path
from scenario_runner import ScenarioRunner

@click.group()
def cli():
    """Integration Test Runner for Library Client"""
    pass

@cli.command()
@click.option('--api-url', required=True, help='API endpoint URL')
@click.option('--scenario', 'scenario_files', multiple=True, help='Scenario files to run')
@click.option('--scenario-dir', help='Directory with scenario files')
def run_scenarios(api_url, scenario_files, scenario_dir):
    """Run integration scenarios"""
    runner = ScenarioRunner(api_url)
    
    # Znajdź wszystkie scenariusze
    all_scenarios = []
    if scenario_files:
        all_scenarios.extend(scenario_files)
    if scenario_dir:
        scenario_path = Path(scenario_dir)
        all_scenarios.extend(scenario_path.glob("*.yaml"))
        all_scenarios.extend(scenario_path.glob("*.yml"))
    
    results = []
    for scenario_file in all_scenarios:
        result = runner.run_scenario(scenario_file)
        results.append(result)
    
    # Podsumowanie
    passed = sum(1 for r in results if r['success'])
    print(f"\n🎯 Summary: {passed}/{len(results)} scenarios passed")

@cli.command()
@click.argument('scenario_template')
def create_scenario(scenario_template):
    """Create new scenario from template"""
    templates = {
        'auth': 'authentication_template.yaml',
        'data': 'data_sync_template.yaml', 
        'error': 'error_handling_template.yaml'
    }
    
    if scenario_template in templates:
        template_file = Path(__file__).parent / 'templates' / templates[scenario_template]
        # Kopiuj template do katalogu scenariuszy
        # ... implementacja
        print(f"📝 Created new scenario from template: {scenario_template}")

if __name__ == '__main__':
    cli()
```

### **5. Przykład użycia przez testerów**

```bash
# 1. Instalacja (tylko raz)
pip install -r requirements.txt

# 2. Uruchomienie pojedynczego scenariusza
python -m runners.cli run-scenarios \
  --api-url https://api-staging.yourservice.com \
  --scenario scenarios/authentication_flow.yaml

# 3. Uruchomienie wszystkich scenariuszy z katalogu
python -m runners.cli run-scenarios \
  --api-url https://api-staging.yourservice.com \
  --scenario-dir scenarios/

# 4. Tworzenie nowego scenariusza
python -m runners.cli create-scenario auth
```

## 🎨 **Dodatkowe korzyści**

### **Dla testerów:**
- **Brak konieczności znajomości C++**
- **Scenariusze w czytelnym YAML**
- **Możliwość tworzenia złożonych flow**
- **Raporty z screenshotami (jeśli potrzeba)**

### **Dla developerów:**
- **Bezpośrednie testowanie biblioteki C++**
- **Możliwość rozbudowy bindingów**
- **Integracja z CI/CD**
- **Debugowanie na poziomie C++**

### **Rozszerzenia:**
```yaml
# Zaawansowany scenariusz z warunkami i pętlami
steps:
  - name: "Load testing"
    action: "repeat"
    parameters:
      times: 100
      steps:
        - name: "Send message {{iteration}}"
          action: "send_data"
          parameters:
            message: "Test message {{iteration}}"
```

Czy ta architektura odpowiada Twoim potrzebom? Chciałbyś, żebym rozwinął któryś element, np. bindingi C++ lub system raportowania?
