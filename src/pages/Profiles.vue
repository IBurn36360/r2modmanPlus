<template>
    <div>
        <!-- Create modal -->
        <div :class="['modal', {'is-active':(addingProfile !== false)}]">
            <div class="modal-background" @click="closeNewProfileModal()"></div>
            <div class='modal-content'>
                 <div class='card'>
                    <header class="card-header">
                        <p class='card-header-title'>{{addingProfileType}} a new profile</p>
                    </header>
                    <div class='card-content'>
                        <p>This profile will store its own mods independently from other profiles.</p>
                        <br/>
                        <input class='input' v-model='newProfileName'/>
                        <br/><br/>
                        <span class="tag is-dark" v-if="newProfileName === '' || makeProfileNameSafe(newProfileName) === ''">Profile name required</span>
                        <span class="tag is-success" v-else-if='!doesProfileExist(newProfileName)'>"{{makeProfileNameSafe(newProfileName)}}" is available</span>
                        <span class="tag is-danger" v-else-if='doesProfileExist(newProfileName)'>"{{makeProfileNameSafe(newProfileName)}}" is already in use</span>
                    </div>
                    <div class='card-footer'>
                        <button class="button is-info" @click="createProfile(newProfileName)">Create</button>
                    </div>
                </div>
            </div>
            <button class="modal-close is-large" aria-label="close" @click="closeNewProfileModal()"></button>
        </div>
        <!-- Delete modal -->
        <div :class="['modal', {'is-active':(removingProfile !== false)}]">
            <div class="modal-background" @click="closeRemoveProfileModal()"></div>
            <div class='modal-content'>
                 <div class='card'>
                    <header class="card-header">
                        <p class='card-header-title'>Delete profile</p>
                    </header>
                    <div class='card-content'>
                        <p>This will remove all mods, and their config files, installed within this profile.</p>
                        <p>If this was an accident, click either the darkened area, or the cross inside located in the top right.</p>
                        <p>Are you sure you'd like to delete this profile?</p>
                    </div>
                    <div class='card-footer'>
                        <button class="button is-danger" @click="removeProfileAfterConfirmation()">Delete profile</button>
                    </div>
                </div>
            </div>
            <button class="modal-close is-large" aria-label="close" @click="closeRemoveProfileModal()"></button>
        </div>
        <!-- Import modal -->
        <div :class="['modal', {'is-active':(importingProfile !== false)}]">
            <div class="modal-background"></div>
            <div class='modal-content'>
                 <div class='card'>
                    <header class="card-header">
                        <p class='card-header-title'>Importing profile</p>
                    </header>
                    <div class='card-content'>
                        <p>This may take a while, as mods are being downloaded.</p>
                        <p>Please do not close r2modman.</p>
                    </div>
                </div>
            </div>
            <button class="modal-close is-large" aria-label="close" @click="closeRemoveProfileModal()"></button>
        </div>
        <!-- Content -->
        <hero title='Profile selection' subtitle='Profiles help to organise mods easily' heroType='is-info' />
        <div class='columns'>
            <div class='column is-full'>
                <div>
                    <article class='media'>
                        <div class='media-content'>
                            <div class='content'>
                                <div v-for='(profileName) of profileList' :key="profileName">
                                    <a @click="selectProfile(profileName)">
                                        <div class='container'>
                                            <div class='border-at-bottom'>
                                                <div class='card is-shadowless'>
                                                    <p :class="['card-header-title', {'has-text-info':selectedProfile === profileName}]">{{profileName}}</p>
                                                </div>
                                            </div>
                                        </div>
                                    </a>
                                </div>
                            </div>
                            <nav class='level'>
                                <div class='level-item'>
                                    <a class='button is-info' @click="setProfileAndContinue()">Use selected profile</a>
                                </div>
                                <div class='level-item'>
                                    <a class='button' @click="newProfile('Create')">Create new</a>
                                </div>
                                <div class='level-item'>
                                    <a class='button' @click="importProfile()">Import profile</a>
                                </div>
                                <div class='level-item'>
                                    <a class='button is-danger' @click="removeProfile()">Delete selected profile</a>
                                </div>
                            </nav>
                        </div>
                    </article>
                </div>
            </div>
        </div>
    </div>
</template>

<script lang='ts'>
import Vue from 'vue';
import Component from 'vue-class-component';
import { Hero, Progress } from '../components/all';
import { Prop } from 'vue-property-decorator';
import { isUndefined, isNull } from 'util';
import sanitize from 'sanitize-filename';
import { ipcRenderer } from 'electron';

import Profile from '../model/Profile';
import VersionNumber from '../model/VersionNumber';
import ThunderstoreMod from '../model/ThunderstoreMod';
import ThunderstoreVersion from '../model/ThunderstoreVersion';
import ManifestV2 from '../model/ManifestV2';
import ExportFormat from '../model/exports/ExportFormat';
import ExportMod from '../model/exports/ExportMod';
import R2Error from '../model/errors/R2Error';
import StatusEnum from '../model/enums/StatusEnum';
import ManagerSettings from '../r2mm/manager/ManagerSettings';
import ThunderstoreDownloader from '../r2mm/downloading/ThunderstoreDownloader';
import ThunderstorePackages from '../r2mm/data/ThunderstorePackages';
import ProfileModList from '../r2mm/mods/ProfileModList';
import ProfileInstaller from '../r2mm/installing/ProfileInstaller';

import * as yaml from 'yaml';
import * as fs from 'fs-extra';
import * as path from 'path';

let settings: ManagerSettings;

@Component({
    components: {
        'hero': Hero,
        'progress-bar': Progress,
    }
})
export default class Profiles extends Vue {
    
    private profileList: string[] = ['Default'];

    private selectedProfile: string = '';

    private addingProfile: boolean = false;
    private newProfileName: string = '';
    private addingProfileType: string = 'Create';

    private removingProfile: boolean = false;
    private importingProfile: boolean = false;

    doesProfileExist(nameToCheck: string): boolean {
        const safe: string | undefined = sanitize(nameToCheck);
        if (isUndefined(safe)) {
            return true;
        }
        return !isUndefined(this.profileList.find((profile: string) => profile.toLowerCase() === safe.toLowerCase()));
    }

    selectProfile(profile: string) {
        new Profile(profile);
        this.selectedProfile = profile;
        settings.setProfile(profile);
    }

    newProfile(type: string, nameOverride: string | undefined) {
        this.newProfileName = nameOverride || '';
        this.addingProfile = true;
        this.addingProfileType = type;
    }

    createProfile(profile: string) {
        const safeName = this.makeProfileNameSafe(profile);
        if (safeName === '') {
            return;
        }
        new Profile(safeName);
        this.profileList.push(safeName);
        this.selectedProfile = Profile.getActiveProfile().getProfileName();
        this.addingProfile = false;
        ipcRenderer.emit('created-profile', safeName);
    }

    closeNewProfileModal() {
        this.addingProfile = false;
        ipcRenderer.emit('created-profile', '');
    }

    removeProfile() {
        this.removingProfile = true;
    }

    removeProfileAfterConfirmation() {
        fs.emptyDirSync(Profile.getActiveProfile().getPathOfProfile());
        fs.removeSync(Profile.getActiveProfile().getPathOfProfile());
        if (Profile.getActiveProfile().getProfileName().toLowerCase() !== 'default') {
            for(let profileIteration = 0; profileIteration < this.profileList.length; profileIteration++) {
                if (this.profileList[profileIteration] === Profile.getActiveProfile().getProfileName()) {
                    this.profileList.splice(profileIteration, 1);
                    break;
                }
            }
        }
        new Profile('Default');
        this.selectedProfile = Profile.getActiveProfile().getProfileName();
        this.closeRemoveProfileModal();
    }

    closeRemoveProfileModal() {
        this.removingProfile = false;
    }

    makeProfileNameSafe(nameToSanitize: string): string {
        const safe: string | undefined = sanitize(nameToSanitize);
        if (isUndefined(safe)) {
            return '';
        }
        return safe;
    }

    setProfileAndContinue() {
        settings.setProfile(Profile.getActiveProfile().getProfileName());
        this.$router.push({path: '/manager'});
    }

    downloadImportedProfileMods(modList: ExportMod[]) {
        let step = 0;
        let currentMod: ThunderstoreMod;
        const installStep = (progress: number, status: number, error: R2Error | void)=>{
            if (status === StatusEnum.SUCCESS) {
                const thunderstoreVersion: ThunderstoreVersion | undefined = currentMod.getVersions().find((version: ThunderstoreVersion) => 
                    version.getVersionNumber().toString() === modList[step].getVersionNumber().toString()
                );
                if (isUndefined(thunderstoreVersion)) {
                    this.importingProfile = false;
                    return;
                }
                const mod: ManifestV2 = new ManifestV2().fromThunderstoreMod(currentMod, thunderstoreVersion);
                ProfileModList.addMod(mod);
                ProfileInstaller.installMod(mod);
                if (!modList[step].isEnabled()) {
                    const profileErr = ProfileInstaller.disableMod(mod);
                    const update: ManifestV2[] | R2Error = ProfileModList.updateMod(mod, (updatingMod: ManifestV2) => {
                        updatingMod.disable();
                    });
                }
                step += 1;
                if (step < modList.length) {
                    const tsMod: ThunderstoreMod | undefined = ThunderstorePackages.PACKAGES.find((mod: ThunderstoreMod) => mod.getFullName() === modList[step].getName());
                    if (isUndefined(tsMod)) {
                        this.importingProfile = false;
                        return;
                    }
                    currentMod = tsMod;
                    const downloader = new ThunderstoreDownloader(tsMod)
                    downloader.download(installStep, modList[step].getVersionNumber());
                } else {
                    this.importingProfile = false;
                }
            } else if (status === StatusEnum.FAILURE) {
                this.importingProfile = false;
                return;
            }
        }
        if (modList.length > 0) {
            const tsMod: ThunderstoreMod | undefined = ThunderstorePackages.PACKAGES.find((mod: ThunderstoreMod) => mod.getFullName() === modList[step].getName());
            if (!(tsMod instanceof ThunderstoreMod)) {
                this.importingProfile = false;
                return;
            }
            currentMod = tsMod;
            const downloader = new ThunderstoreDownloader(tsMod);
            downloader.download(installStep, modList[step].getVersionNumber());
        }
    }

    importProfile(file: string) {
        ipcRenderer.once('receive-selection', (_sender: any, files: string[] | null) => {
            if (isNull(files) || files.length === 0) {
                this.importingProfile = false;
                return;
            }
            const read: string = fs.readFileSync(files[0]).toString();
            const parsedYaml = yaml.parse(read);
            const parsed: ExportFormat = new ExportFormat(parsedYaml.profileName, parsedYaml.mods.map((mod: any) => {
                const enabled = isUndefined(mod.enabled) || mod.enabled;
                return new ExportMod(mod.name, new VersionNumber(`${mod.version.major}.${mod.version.minor}.${mod.version.patch}`), enabled);
            }));
            this.newProfile('Import', parsed.getProfileName());
            ipcRenderer.once('created-profile', (profileName: string) => {
                if (profileName !== '') {
                    this.importingProfile = true;
                    this.downloadImportedProfileMods(parsed.getMods());
                }
            })
        });
        ipcRenderer.send('open-dialog', {
            title: 'Import Profile',
            properties: ['openFile'],
            filters: ['.r2x'],
            buttonLabel: 'Import',
        });
    }

    created() {
        settings = new ManagerSettings();
        settings.load();

        this.selectedProfile = settings.lastSelectedProfile;
        new Profile(this.selectedProfile);

        try {
            const profilesDirectory: string = Profile.getActiveProfile().getDirectory();
            fs.readdirSync(profilesDirectory)
                .forEach((file: string) => {
                    if (fs.lstatSync(path.join(profilesDirectory, file)).isDirectory() && file.toLowerCase() !== 'default') {
                        this.profileList.push(file);
                    }
                })
        } catch(e) {
            return;
        }
    }

}
</script>
