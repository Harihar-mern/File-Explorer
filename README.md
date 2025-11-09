# File-Explorer
// =============================
// 📂 FILE: file_explorer.cpp
// =============================
#include <bits/stdc++.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <dirent.h>
#include <unistd.h>
#include <pwd.h>
#include <grp.h>
#include <fcntl.h>
#include <utime.h>
using namespace std;

string get_permissions(mode_t mode) {
    string perms;
    perms += (S_ISDIR(mode)) ? 'd' : '-';
    perms += (mode & S_IRUSR) ? 'r' : '-';
    perms += (mode & S_IWUSR) ? 'w' : '-';
    perms += (mode & S_IXUSR) ? 'x' : '-';
    perms += (mode & S_IRGRP) ? 'r' : '-';
    perms += (mode & S_IWGRP) ? 'w' : '-';
    perms += (mode & S_IXGRP) ? 'x' : '-';
    perms += (mode & S_IROTH) ? 'r' : '-';
    perms += (mode & S_IWOTH) ? 'w' : '-';
    perms += (mode & S_IXOTH) ? 'x' : '-';
    return perms;
}

void list_files_simple(const string& path) {
    DIR* dir = opendir(path.c_str());
    if (!dir) {
        perror("Error opening directory");
        return;
    }
    dirent* entry;
    while ((entry = readdir(dir)) != NULL)
        cout << entry->d_name << endl;
    closedir(dir);
}

void list_files_detailed(const string& path) {
    DIR* dir = opendir(path.c_str());
    if (!dir) {
        perror("Error opening directory");
        return;
    }
    dirent* entry;
    cout << left << setw(12) << "Permissions" << setw(10) << "Owner"
         << setw(10) << "Group" << setw(10) << "Size" << "Modified" << endl;
    cout << string(60, '-') << endl;

    while ((entry = readdir(dir)) != NULL) {
        string filepath = path + "/" + entry->d_name;
        struct stat fileStat;
        if (stat(filepath.c_str(), &fileStat) < 0) continue;
        struct passwd *pw = getpwuid(fileStat.st_uid);
        struct group *gr = getgrgid(fileStat.st_gid);
        string timeStr = ctime(&fileStat.st_mtime);
        timeStr.pop_back();
        cout << setw(12) << get_permissions(fileStat.st_mode)
             << setw(10) << pw->pw_name
             << setw(10) << gr->gr_name
             << setw(10) << fileStat.st_size
             << " " << timeStr << "  " << entry->d_name << endl;
    }
    closedir(dir);
}

void create_file(const string& filename) {
    ofstream file(filename);
    if (file) cout << "File created successfully.\n";
    else perror("Error creating file");
}

void create_directory(const string& dirname) {
    if (mkdir(dirname.c_str(), 0755) == 0) cout << "Directory created.\n";
    else perror("Error creating directory");
}

void delete_file_or_dir(const string& path) {
    struct stat s;
    if (stat(path.c_str(), &s) == 0) {
        if (S_ISDIR(s.st_mode)) {
            string cmd = "rm -r '" + path + "'";
            system(cmd.c_str());
        } else unlink(path.c_str());
        cout << "Deleted successfully.\n";
    } else perror("Error deleting");
}

void copy_file(const string& src, const string& dest) {
    ifstream in(src, ios::binary);
    ofstream out(dest, ios::binary);
    out << in.rdbuf();
    cout << "File copied successfully.\n";
}

void move_file(const string& src, const string& dest) {
    if (rename(src.c_str(), dest.c_str()) == 0)
        cout << "Moved successfully.\n";
    else perror("Error moving");
}

void rename_file(const string& oldn, const string& newn) {
    if (rename(oldn.c_str(), newn.c_str()) == 0)
        cout << "Renamed successfully.\n";
    else perror("Error renaming");
}

void search_files(const string& base, const string& term) {
    DIR* dir = opendir(base.c_str());
    if (!dir) return;
    dirent* entry;
    while ((entry = readdir(dir)) != NULL) {
        if (strcmp(entry->d_name, ".") == 0 || strcmp(entry->d_name, "..") == 0)
            continue;
        string path = base + "/" + entry->d_name;
        if (strcasestr(entry->d_name, term.c_str()))
            cout << path << endl;
        struct stat s;
        if (stat(path.c_str(), &s) == 0 && S_ISDIR(s.st_mode))
            search_files(path, term);
    }
    closedir(dir);
}

void view_permissions(const string& filename) {
    struct stat fileStat;
    if (stat(filename.c_str(), &fileStat) < 0) {
        perror("Error getting permissions");
        return;
    }
    cout << "Permissions: " << get_permissions(fileStat.st_mode) << endl;
    cout << "Owner: " << getpwuid(fileStat.st_uid)->pw_name << endl;
    cout << "Group: " << getgrgid(fileStat.st_gid)->gr_name << endl;
    cout << "Size: " << fileStat.st_size << " bytes\n";
    cout << "Last Modified: " << ctime(&fileStat.st_mtime);
}

void change_permissions(const string& filename, const string& perm) {
    mode_t mode = stoi(perm, nullptr, 8);
    if (chmod(filename.c_str(), mode) == 0)
        cout << "Permissions changed.\n";
    else perror("Error changing permissions");
}

void change_owner(const string& filename, const string& user, const string& group) {
    struct passwd* pw = getpwnam(user.c_str());
    struct group* gr = getgrnam(group.c_str());
    if (!pw || !gr) {
        cerr << "Invalid user/group\n";
        return;
    }
    if (chown(filename.c_str(), pw->pw_uid, gr->gr_gid) == 0)
        cout << "Owner/Group changed.\n";
    else perror("Error changing owner");
}

int main() {
    while (true) {
        char cwd[1024];
        getcwd(cwd, sizeof(cwd));
        cout << "\n==================== FILE EXPLORER MENU ====================\n";
        cout << "Current Directory: " << cwd << "\n";
        cout << "------------------------------------------------------------\n";
        cout << "1.  List files (simple)\n2.  List files (detailed)\n3.  Change directory\n4.  Go to parent directory\n";
        cout << "5.  Create file\n6.  Create directory\n7.  Delete file/directory\n8.  Copy file\n9.  Move file\n";
        cout << "10. Rename file/directory\n11. Search files\n12. View file permissions\n13. Change permissions (chmod)\n";
        cout << "14. Change owner/group (chown)\n15. Display current path\n0.  Exit\n";
        cout << "------------------------------------------------------------\n";
        cout << "Choose an option: ";
        int ch; cin >> ch;
        cin.ignore();
        if (ch == 0) break;

        string a, b;
        switch (ch) {
            case 1: list_files_simple(cwd); break;
            case 2: list_files_detailed(cwd); break;
            case 3: cout << "Enter directory: "; getline(cin, a); chdir(a.c_str()); break;
            case 4: chdir(".."); break;
            case 5: cout << "Enter filename: "; getline(cin, a); create_file(a); break;
            case 6: cout << "Enter directory: "; getline(cin, a); create_directory(a); break;
            case 7: cout << "Enter path: "; getline(cin, a); delete_file_or_dir(a); break;
            case 8: cout << "Enter source: "; getline(cin, a); cout << "Enter destination: "; getline(cin, b); copy_file(a, b); break;
            case 9: cout << "Enter source: "; getline(cin, a); cout << "Enter destination: "; getline(cin, b); move_file(a, b); break;
            case 10: cout << "Enter current name: "; getline(cin, a); cout << "Enter new name: "; getline(cin, b); rename_file(a, b); break;
            case 11: cout << "Enter search term: "; getline(cin, a); search_files(".", a); break;
            case 12: cout << "Enter filename: "; getline(cin, a); view_permissions(a); break;
            case 13: cout << "Enter filename: "; getline(cin, a); cout << "Enter octal (e.g., 755): "; getline(cin, b); change_permissions(a, b); break;
            case 14: cout << "Enter filename: "; getline(cin, a); cout << "Enter new owner: "; getline(cin, b); {string g; cout << "Enter group: "; getline(cin, g); change_owner(a, b, g);} break;
            case 15: cout << "Current path: " << cwd << endl; break;
            default: cout << "Invalid option.\n"; break;
        }
    }
    cout << "Exiting File Explorer.\n";
    return 0;
}
