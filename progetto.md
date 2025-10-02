Perfetto 😊
hai ragione: posso stamparti subito tutto il progetto completo in un colpo solo.
Ecco quindi il toolkit Ansible pronto all’uso, già in formato YAML.

📁 Struttura del progetto
```
config_toolkit/
├── apply_changes.yml
├── rollback_changes.yml
├── changes_to_apply.yml
├── data/        # conterrà i file di rollback generati automaticamente
└── logs/        # conterrà i log delle esecuzioni
```

1️⃣ changes_to_apply.yml

Il file di input con tutte le modifiche che vuoi applicare.
```
# changes_to_apply.yml
changes:
  - host: server01
    modifications:
      - path: /etc/sysctl.conf
        original_line: "net.ipv4.ip_forward = 0"
        updated_line: "net.ipv4.ip_forward = 1"
      - path: /etc/security.conf
        original_line: "ALLOW_SSH=no"
        updated_line: "ALLOW_SSH=yes"

  - host: server02
    modifications:
      - path: /opt/custom/app.conf
        original_line: "feature_enabled=false"
        updated_line: "feature_enabled=true"
```

2️⃣ apply_changes.yml

Playbook per applicare modifiche e creare automaticamente il file di rollback + log.
```
# apply_changes.yml
- name: Applica modifiche e genera rollback file
  hosts: all
  gather_facts: false
  vars_files:
    - changes_to_apply.yml
  vars_prompt:
    - name: label
      prompt: "Inserisci label per questa modifica"
      private: no

  tasks:
    - name: Recupera le modifiche per questo host
      set_fact:
        host_changes: "{{ changes | selectattr('host', 'equalto', inventory_hostname) | list | first }}"
      when: changes | selectattr('host', 'equalto', inventory_hostname) | list | length > 0

    - name: Fallisci se non ci sono modifiche per questo host
      fail:
        msg: "Nessuna modifica registrata per {{ inventory_hostname }}"
      when: host_changes is not defined

    - name: Applica modifiche per questo host
      loop: "{{ host_changes.modifications }}"
      loop_control:
        label: "{{ item.path }}"
      ansible.builtin.replace:
        path: "{{ item.path }}"
        regexp: "{{ item.original_line | regex_escape() }}"
        replace: "{{ item.updated_line }}"
        backup: yes

    - name: Crea il file di rollback sul controller
      delegate_to: localhost
      run_once: true
      copy:
        dest: "data/rollback_{{ label }}.yml"
        content: |
          rollback_label: "{{ label }}"
          changes:
          {% for ch in changes %}
            - host: {{ ch.host }}
              modifications:
              {% for m in ch.modifications %}
                - path: {{ m.path }}
                  original_line: "{{ m.updated_line }}"
                  updated_line: "{{ m.original_line }}"
              {% endfor %}
          {% endfor %}

    - name: Scrivi log sul controller
      delegate_to: localhost
      run_once: true
      copy:
        dest: "logs/apply_{{ label }}.log"
        content: "Modifiche applicate con label {{ label }} il {{ ansible_date_time.date }} alle {{ ansible_date_time.time }}\n"
```

3️⃣ rollback_changes.yml

Playbook per rollback selezionando il file rollback_<label>.yml in data/.
```
# rollback_changes.yml
- name: Esegui rollback delle modifiche
  hosts: all
  gather_facts: false
  vars_prompt:
    - name: label
      prompt: "Inserisci la label del rollback da applicare"
      private: no
  vars_files:
    - "data/rollback_{{ label }}.yml"

  tasks:
    - name: Recupera le modifiche di rollback per questo host
      set_fact:
        host_changes: "{{ changes | selectattr('host', 'equalto', inventory_hostname) | list | first }}"
      when: changes | selectattr('host', 'equalto', inventory_hostname) | list | length > 0

    - name: Fallisci se non ci sono modifiche per questo host
      fail:
        msg: "Nessuna modifica di rollback registrata per {{ inventory_hostname }}"
      when: host_changes is not defined

    - name: Esegui il rollback delle modifiche per questo host
      loop: "{{ host_changes.modifications }}"
      loop_control:
        label: "{{ item.path }}"
      ansible.builtin.replace:
        path: "{{ item.path }}"
        regexp: "{{ item.original_line | regex_escape() }}"
        replace: "{{ item.updated_line }}"
        backup: yes

    - name: Crea il file di rollback del rollback (inversione di nuovo)
      delegate_to: localhost
      run_once: true
      copy:
        dest: "data/rollback_of_{{ label }}.yml"
        content: |
          rollback_label: "rollback_of_{{ label }}"
          changes:
          {% for ch in changes %}
            - host: {{ ch.host }}
              modifications:
              {% for m in ch.modifications %}
                - path: {{ m.path }}
                  original_line: "{{ m.updated_line }}"
                  updated_line: "{{ m.original_line }}"
              {% endfor %}
          {% endfor %}

    - name: Scrivi log sul controller
      delegate_to: localhost
      run_once: true
      copy:
        dest: "logs/rollback_{{ label }}.log"
        content: "Rollback eseguito con label {{ label }} il {{ ansible_date_time.date }} alle {{ ansible_date_time.time }}\n"
```

🔧 Come usarli

Applicare modifiche:

ansible-playbook -i inventory apply_changes.yml


(ti chiederà una label)

Rollback modifiche:

(ti chiederà una label)

ansible-playbook -i inventory rollback_changes.yml
(ti chiederà la label del rollback da usare, es: fix_sysctl)

✅ Cosa fa questo sistema:

Salva ogni file di rollback in data/rollback_<label>.yml

Salva ogni log in logs/apply_<label>.log o logs/rollback_<label>.log

Al rollback, crea anche rollback_of_<label>.yml per permettere il rollback del rollback

Usa solo moduli Ansible built-in (niente shell/grep)

Vuoi che aggiunga anche hash dei file originali per maggiore sicurezza (così verifica se il file è stato cambiato esternamente prima di modificare/rollbackare)?
