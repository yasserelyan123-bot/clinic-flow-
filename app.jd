// ============================================================
// ClinicFlow — التحديث الكبير: تصميم أفضل + وحدات كاملة
// ============================================================
// يستخدم fetch() فقط للتواصل مع Supabase (بدون أي مكتبة خارجية)،
// نفس الأساس اللي أثبت إنه شغّال، مع إضافة واجهات فعلية لوحدة
// الحمل ووحدة الأمراض المزمنة (بما فيها السرطان، عبر حقل نصي حر
// لا يحتاج أي تعديل في قاعدة البيانات) والمواعيد.
// ============================================================

window.__clinicflow_started__ = true;

const API_BASE = window.SUPABASE_URL;
const ANON_KEY = window.SUPABASE_ANON_KEY;
const STORAGE_KEY = 'clinicflow_session';

const COMMON_CONDITIONS = [
    'السكري (Type 2 Diabetes)',
    'ضغط الدم المرتفع (Hypertension)',
    'السرطان (Cancer)',
    'أمراض القلب',
    'الفشل الكلوي',
    'الربو',
    'الغدة الدرقية',
];

function showStartupError(title, detail) {
    const el = document.getElementById('app');
    if (el) {
        el.innerHTML = `
            <div style="max-width:600px;margin:40px auto;padding:20px;background:#fee;border:2px solid #f88;border-radius:8px;font-family:sans-serif;direction:rtl;text-align:right;">
                <h2 style="color:#c00;margin-top:0;">${title}</h2>
                <p style="color:#600;white-space:pre-wrap;">${detail}</p>
            </div>`;
    }
}

function showMessage(text, type = 'info') {
    const colors = {
        error: { bg: '#fee2e2', border: '#f87171', text: '#991b1b' },
        success: { bg: '#dcfce7', border: '#4ade80', text: '#166534' },
        info: { bg: '#dbeafe', border: '#60a5fa', text: '#1e40af' },
    };
    const c = colors[type] || colors.info;
    let banner = document.getElementById('clinicflow-message-banner');
    if (!banner) {
        banner = document.createElement('div');
        banner.id = 'clinicflow-message-banner';
        banner.style.cssText = `position:sticky;top:0;z-index:9999;margin:0;padding:14px 20px;font-family:'Cairo',sans-serif;direction:rtl;text-align:right;font-size:15px;line-height:1.5;`;
        document.body.prepend(banner);
    }
    banner.style.background = c.bg;
    banner.style.borderBottom = `2px solid ${c.border}`;
    banner.style.color = c.text;
    banner.textContent = text;
    banner.style.display = 'block';
    clearTimeout(window.__clinicflow_msg_timeout__);
    window.__clinicflow_msg_timeout__ = setTimeout(() => { banner.style.display = 'none'; }, 6000);
}

// ---------- طبقة اتصال خام بـ Supabase ----------

async function authRequest(path, body) {
    const res = await fetch(`${API_BASE}/auth/v1/${path}`, {
        method: 'POST',
        headers: { 'apikey': ANON_KEY, 'Content-Type': 'application/json' },
        body: JSON.stringify(body),
    });
    const data = await res.json().catch(() => ({}));
    if (!res.ok) {
        throw new Error(data.error_description || data.msg || data.error || 'حدث خطأ في المصادقة');
    }
    return data;
}

async function getAuthUser(accessToken) {
    const res = await fetch(`${API_BASE}/auth/v1/user`, {
        headers: { 'apikey': ANON_KEY, 'Authorization': 'Bearer ' + accessToken },
    });
    if (!res.ok) return null;
    return res.json();
}

async function pgFetch(path, { method = 'GET', body, accessToken, extraHeaders = {} } = {}) {
    const res = await fetch(`${API_BASE}/rest/v1/${path}`, {
        method,
        headers: {
            'apikey': ANON_KEY,
            'Authorization': 'Bearer ' + accessToken,
            'Content-Type': 'application/json',
            ...extraHeaders,
        },
        body: body !== undefined ? JSON.stringify(body) : undefined,
    });
    const text = await res.text();
    const data = text ? JSON.parse(text) : null;
    if (!res.ok) {
        const msg = (data && (data.message || data.error_description || data.msg)) || 'حدث خطأ في الطلب';
        throw new Error(msg);
    }
    return data;
}

function pgRpc(fnName, params, accessToken) {
    return pgFetch(`rpc/${fnName}`, { method: 'POST', body: params, accessToken });
}

function saveSession(session) { localStorage.setItem(STORAGE_KEY, JSON.stringify(session)); }
function loadStoredSession() {
    try { const raw = localStorage.getItem(STORAGE_KEY); return raw ? JSON.parse(raw) : null; }
    catch { return null; }
}
function clearSession() { localStorage.removeItem(STORAGE_KEY); }

function fmtDate(d) { return d ? new Date(d).toLocaleDateString('ar-EG') : '-'; }
function esc(s) { return (s ?? '').toString().replace(/[&<>"']/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c])); }

// ---------- التطبيق ----------

class ClinicApp {
    constructor() {
        this.session = null;
        this.profile = null;
        this.currentPage = 'login';
        this.selectedPatient = null;
        this._patientsCache = [];
        this.init().catch((e) => {
            console.error(e);
            showStartupError('حدث خطأ أثناء تحميل التطبيق', e.message || String(e));
        });
    }

    async init() {
        const stored = loadStoredSession();
        if (stored && stored.access_token) {
            const user = await getAuthUser(stored.access_token);
            if (user) {
                this.session = { ...stored, user };
            } else if (stored.refresh_token) {
                try {
                    const refreshed = await authRequest('token?grant_type=refresh_token', { refresh_token: stored.refresh_token });
                    this.session = refreshed;
                    saveSession(refreshed);
                } catch { clearSession(); }
            } else {
                clearSession();
            }
        }
        if (this.session) {
            await this.loadProfile();
            if (this.profile) await this.loadClinic();
        }
        this.currentPage = this.profile ? 'dashboard' : (this.session ? 'complete-signup' : 'login');
        this.render();
        if (this.currentPage === 'dashboard') this.loadDashboard();
    }

    async loadProfile() {
        try {
            const rows = await pgFetch(`profiles?id=eq.${this.session.user.id}&select=*`, { accessToken: this.session.access_token });
            this.profile = rows && rows[0] ? rows[0] : null;
        } catch (e) {
            console.error(e);
            this.profile = null;
            this._lastProfileError = e.message;
        }
    }

    async loadClinic() {
        try {
            const rows = await pgFetch(`clinics?id=eq.${this.profile.clinic_id}&select=*`, { accessToken: this.token });
            this.clinic = rows && rows[0] ? rows[0] : { specialty: 'general' };
        } catch (e) {
            console.error(e);
            this.clinic = { specialty: 'general' };
        }
    }

    // خرائط التخصص: كل تخصص له لون أساسي وعنوان مختلف للوحة التحكم
    get specialtyTheme() {
        const specialty = this.clinic?.specialty || 'general';
        const themes = {
            obgyn: { color: 'pink', label: 'عيادة النساء والتوليد', icon: '🤰' },
            chronic: { color: 'amber', label: 'عيادة متابعة الأمراض المزمنة', icon: '💊' },
            oncology: { color: 'purple', label: 'عيادة الأورام', icon: '🎗️' },
            general: { color: 'blue', label: 'عيادة عامة', icon: '🏥' },
        };
        return themes[specialty] || themes.general;
    }

    logout() {
        clearSession();
        this.session = null;
        this.profile = null;
        this.currentPage = 'login';
        this.render();
    }

    get token() { return this.session?.access_token; }

    // ---------- Auth ----------
    async login(event) {
        event.preventDefault();
        const form = new FormData(event.target);
        try {
            const session = await authRequest('token?grant_type=password', { email: form.get('email'), password: form.get('password') });
            this.session = session;
            saveSession(session);
            await this.loadProfile();
            if (this.profile) await this.loadClinic();
            this.navigate(this.profile ? 'dashboard' : 'complete-signup');
        } catch (e) { showMessage('فشل تسجيل الدخول: ' + e.message, 'error'); }
    }

    async registerAccount(event) {
        event.preventDefault();
        const form = new FormData(event.target);
        try {
            const result = await authRequest('signup', { email: form.get('email'), password: form.get('password') });
            if (!result.access_token) {
                showMessage('تم إنشاء الحساب. تحقق من بريدك لتأكيد الحساب، ثم سجّل الدخول.', 'info');
                this.navigate('login');
                return;
            }
            this.session = result;
            saveSession(result);
            this._pendingClinicName = form.get('clinic_name');
            this._pendingAdminName = form.get('admin_full_name');
            this._pendingSpecialty = form.get('specialty') || 'general';
            await this.finishClinicRegistration();
        } catch (e) { showMessage('فشل إنشاء الحساب: ' + e.message, 'error'); }
    }

    async completeSignup(event) {
        event.preventDefault();
        const form = new FormData(event.target);
        this._pendingClinicName = form.get('clinic_name');
        this._pendingAdminName = form.get('admin_full_name');
        this._pendingSpecialty = form.get('specialty') || 'general';
        await this.finishClinicRegistration();
    }

    async finishClinicRegistration() {
        try {
            await pgRpc('register_clinic', {
                p_clinic_name: this._pendingClinicName,
                p_admin_full_name: this._pendingAdminName,
                p_specialty: this._pendingSpecialty || 'general',
            }, this.token);
            await this.loadProfile();
            await this.loadClinic();
            this.navigate('dashboard');
        } catch (e) { showMessage('فشل إنشاء العيادة: ' + e.message, 'error'); }
    }

    // ---------- Navigation ----------
    navigate(page, data = null) {
        this.currentPage = page;
        if (data) this.selectedPatient = data;
        this.render();
        if (page === 'dashboard') this.loadDashboard();
        if (page === 'patients') this.loadPatients();
        if (page === 'appointments') this.loadAppointments();
        if (page === 'patient-detail') this.loadPatientDetail();
    }

    // ---------- Dashboard ----------
    async loadDashboard() {
        const token = this.token;
        try {
            const [todayCount, patients, visits, pregnancies] = await Promise.all([
                this._todayAppointmentsCount(token),
                pgFetch('patients?select=id', { accessToken: token }),
                pgFetch('visits?select=id', { accessToken: token }),
                pgFetch('pregnancies?status=eq.active&select=id', { accessToken: token }),
            ]);
            document.getElementById('today-appointments').textContent = todayCount;
            document.getElementById('total-patients').textContent = patients.length;
            document.getElementById('total-visits').textContent = visits.length;
            document.getElementById('active-pregnancies').textContent = pregnancies.length;
        } catch (e) { console.error(e); }

        try {
            const pregnancies = await pgFetch('active_pregnancies_view?select=*&order=edd_date.asc', { accessToken: token });
            let patientsById = {};
            if (pregnancies && pregnancies.length) {
                const ids = pregnancies.map(p => p.patient_id).join(',');
                const patients = await pgFetch(`patients?id=in.(${ids})&select=id,name`, { accessToken: token });
                patientsById = Object.fromEntries(patients.map(p => [p.id, p.name]));
            }
            const el = document.getElementById('active-pregnancies-list');
            if (el) {
                el.innerHTML = (pregnancies && pregnancies.length)
                    ? pregnancies.map(p => `
                        <div class="flex justify-between items-center border-b border-gray-100 py-3">
                            <div>
                                <span class="font-semibold text-gray-800">${esc(patientsById[p.patient_id] || 'مريضة')}</span>
                                <span class="text-sm text-gray-500 mr-2">أسبوع ${p.current_ga_weeks ?? '-'} + ${p.current_ga_days ?? 0} يوم</span>
                            </div>
                            <span class="text-xs bg-pink-50 text-pink-700 px-2 py-1 rounded-full">EDD: ${fmtDate(p.edd_date)}</span>
                        </div>`).join('')
                    : '<p class="text-gray-400 text-center py-6">لا توجد حالات حمل نشطة حاليًا</p>';
            }
        } catch (e) { console.error(e); }

        try {
            const allConditions = await pgFetch('chronic_conditions?status=eq.active&select=*,patients(name)&order=created_at.desc&limit=30', { accessToken: token });
            const isCancer = (name) => /سرطان|cancer|ورم خبيث|أورام/i.test(name || '');
            const chronicOnly = (allConditions || []).filter(c => !isCancer(c.condition_name)).slice(0, 8);
            const oncologyOnly = (allConditions || []).filter(c => isCancer(c.condition_name)).slice(0, 8);

            const renderConditionRow = (c, badgeColor) => `
                <div class="flex justify-between items-center border-b border-gray-100 py-3">
                    <div>
                        <span class="font-semibold text-gray-800">${esc(c.patients?.name || 'مريض')}</span>
                        <span class="text-sm text-gray-500 mr-2">${esc(c.condition_name)}</span>
                    </div>
                    <span class="text-xs bg-${badgeColor}-50 text-${badgeColor}-700 px-2 py-1 rounded-full">${esc(c.status)}</span>
                </div>`;

            const chronicEl = document.getElementById('chronic-conditions-list');
            if (chronicEl) {
                chronicEl.innerHTML = chronicOnly.length
                    ? chronicOnly.map(c => renderConditionRow(c, 'amber')).join('')
                    : '<p class="text-gray-400 text-center py-6">لا توجد حالات مزمنة نشطة</p>';
            }
            const oncologyEl = document.getElementById('oncology-conditions-list');
            if (oncologyEl) {
                oncologyEl.innerHTML = oncologyOnly.length
                    ? oncologyOnly.map(c => renderConditionRow(c, 'purple')).join('')
                    : '<p class="text-gray-400 text-center py-6">لا توجد حالات أورام نشطة</p>';
            }
        } catch (e) { console.error(e); }
    }

    async _todayAppointmentsCount(token) {
        const today = new Date();
        const start = new Date(today.getFullYear(), today.getMonth(), today.getDate()).toISOString();
        const end = new Date(today.getFullYear(), today.getMonth(), today.getDate() + 1).toISOString();
        const rows = await pgFetch(`appointments?select=id&start_time=gte.${start}&start_time=lt.${end}`, { accessToken: token });
        return rows.length;
    }

    // ---------- Patients ----------
    async loadPatients() {
        try {
            const data = await pgFetch('patients?select=*&order=created_at.desc', { accessToken: this.token });
            this._patientsCache = data || [];
            this.renderPatients(data);
        } catch (e) { showMessage('تعذّر تحميل المرضى: ' + e.message, 'error'); }
    }

    async createPatient(event) {
        event.preventDefault();
        const form = new FormData(event.target);
        const token = this.token;
        try {
            const branches = await pgFetch(`branches?clinic_id=eq.${this.profile.clinic_id}&select=id&limit=1`, { accessToken: token });
            const branchId = branches && branches[0] ? branches[0].id : null;
            const payload = {
                clinic_id: this.profile.clinic_id,
                branch_id: branchId,
                name: form.get('name'),
                gender: form.get('gender'),
                phone: form.get('phone'),
                date_of_birth: form.get('date_of_birth') || null,
                allergies: form.get('allergies') || null,
                chronic_conditions: form.get('chronic_conditions') || null,
            };
            const rows = await pgFetch('patients', { method: 'POST', body: payload, accessToken: token, extraHeaders: { 'Prefer': 'return=representation' } });
            showMessage('تم إضافة المريض بنجاح! رقم الملف: ' + rows[0].patient_number, 'success');
            event.target.reset();
            this.loadPatients();
        } catch (e) { showMessage('حدث خطأ: ' + e.message, 'error'); }
    }

    renderPatients(patients) {
        const container = document.getElementById('patients-list');
        if (!container) return;
        if (!patients || patients.length === 0) {
            container.innerHTML = '<p class="text-gray-400 text-center py-10">لا يوجد مرضى بعد — أضف أول مريض من النموذج</p>';
            return;
        }
        container.innerHTML = patients.map(p => `
            <div class="bg-white p-4 rounded-xl shadow-sm border border-gray-100 mb-3 hover:shadow-md transition-shadow">
                <div class="flex justify-between items-center">
                    <div>
                        <h3 class="font-bold text-gray-800">${esc(p.name)}</h3>
                        <p class="text-xs text-gray-500 mt-1">رقم الملف: ${esc(p.patient_number)} · ${esc(p.phone || '-')}</p>
                    </div>
                    <button data-id="${p.id}" class="view-patient-btn bg-blue-600 text-white text-sm px-4 py-2 rounded-lg hover:bg-blue-700">
                        عرض الملف
                    </button>
                </div>
            </div>
        `).join('');
        container.querySelectorAll('.view-patient-btn').forEach(btn => {
            btn.addEventListener('click', () => this.navigate('patient-detail', { id: btn.dataset.id }));
        });
    }

    // ---------- Patient detail ----------
    async loadPatientDetail() {
        const id = this.selectedPatient?.id;
        if (!id) return;
        const token = this.token;

        try {
            const rows = await pgFetch(`patients?id=eq.${id}&select=*`, { accessToken: token });
            this._currentPatient = rows[0];
            if (this._currentPatient) {
                const p = this._currentPatient;
                document.getElementById('patient-detail-header').innerHTML = `
                    <div class="flex justify-between items-start">
                        <div>
                            <h2 class="text-xl font-bold text-gray-800">${esc(p.name)}</h2>
                            <p class="text-sm text-gray-500 mt-1">رقم الملف: ${esc(p.patient_number)} · ${esc(p.phone || '-')} · ${p.gender === 'female' ? 'أنثى' : 'ذكر'}</p>
                            ${p.allergies ? `<p class="text-sm text-red-600 mt-2">⚠️ حساسية: ${esc(p.allergies)}</p>` : ''}
                        </div>
                    </div>`;
            }
        } catch (e) { showMessage('تعذّر تحميل بيانات المريض: ' + e.message, 'error'); }

        this.loadPatientVisits(id, token);
        this.loadPatientPregnancy(id, token);
        this.loadPatientChronic(id, token);
    }

    async loadPatientVisits(id, token) {
        try {
            const visits = await pgFetch(`visits?patient_id=eq.${id}&select=*&order=created_at.desc`, { accessToken: token });
            document.getElementById('patient-visits').innerHTML = (visits && visits.length)
                ? visits.map(v => `
                    <div class="border-b border-gray-100 py-2">
                        <div class="flex justify-between">
                            <span class="font-semibold text-sm">${esc(v.diagnosis || v.chief_complaint)}</span>
                            <span class="text-xs text-gray-400">${fmtDate(v.created_at)}</span>
                        </div>
                        ${v.bp_systolic ? `<p class="text-xs text-gray-500">ضغط: ${v.bp_systolic}/${v.bp_diastolic}</p>` : ''}
                    </div>`).join('')
                : '<p class="text-gray-400 text-sm text-center py-4">لا توجد زيارات مسجلة</p>';
        } catch (e) { console.error(e); }
    }

    // ---------- Pregnancy module ----------
    async loadPatientPregnancy(id, token) {
        const el = document.getElementById('pregnancy-section');
        if (!el) return;
        try {
            const pregnancies = await pgFetch(`pregnancies?patient_id=eq.${id}&status=eq.active&select=*&order=created_at.desc&limit=1`, { accessToken: token });
            const pregnancy = pregnancies && pregnancies[0];
            if (!pregnancy) {
                el.innerHTML = `
                    <p class="text-gray-400 text-sm mb-3">لا توجد متابعة حمل نشطة لهذه المريضة</p>
                    <button id="start-pregnancy-btn" class="w-full bg-pink-500 text-white py-2 rounded-lg text-sm hover:bg-pink-600">+ بدء متابعة حمل جديدة</button>
                    <div id="pregnancy-form-container"></div>`;
                document.getElementById('start-pregnancy-btn').addEventListener('click', () => this.showNewPregnancyForm());
                return;
            }
            this._currentPregnancy = pregnancy;
            const lmp = pregnancy.lmp_date ? new Date(pregnancy.lmp_date) : null;
            let gaWeeks = '-', gaDays = '-';
            if (lmp) {
                const days = Math.floor((new Date() - lmp) / 86400000);
                if (days >= 0) { gaWeeks = Math.floor(days / 7); gaDays = days % 7; }
            }
            const ancVisits = await pgFetch(`anc_visits?pregnancy_id=eq.${pregnancy.id}&select=*&order=visit_date.desc`, { accessToken: token });
            el.innerHTML = `
                <div class="bg-pink-50 rounded-lg p-4 mb-4">
                    <div class="flex justify-between items-center">
                        <span class="font-bold text-pink-800">أسبوع ${gaWeeks} + ${gaDays} يوم</span>
                        <span class="text-xs text-pink-600">EDD: ${fmtDate(pregnancy.edd_date)}</span>
                    </div>
                    <p class="text-xs text-gray-600 mt-1">G${pregnancy.gravida ?? '-'} P${pregnancy.para ?? '-'} A${pregnancy.abortions ?? '-'} ${pregnancy.is_multiple ? '· حمل متعدد' : ''}</p>
                    ${pregnancy.risk_factors ? `<p class="text-xs text-red-600 mt-1">⚠️ ${esc(pregnancy.risk_factors)}</p>` : ''}
                </div>
                <button id="add-anc-btn" class="w-full bg-pink-500 text-white py-2 rounded-lg text-sm hover:bg-pink-600 mb-3">+ تسجيل زيارة متابعة (ANC)</button>
                <div id="anc-form-container"></div>
                <h4 class="text-sm font-bold text-gray-700 mt-3 mb-2">سجل زيارات المتابعة</h4>
                <div>${(ancVisits && ancVisits.length) ? ancVisits.map(v => `
                    <div class="border-b border-gray-100 py-2 text-sm">
                        <div class="flex justify-between">
                            <span>أسبوع ${v.gestational_age_weeks ?? '-'}</span>
                            <span class="text-xs text-gray-400">${fmtDate(v.visit_date)}</span>
                        </div>
                        ${v.bp_systolic ? `<p class="text-xs text-gray-500">ضغط: ${v.bp_systolic}/${v.bp_diastolic}${v.fetal_heart_rate ? ' · نبض الجنين: ' + v.fetal_heart_rate : ''}</p>` : ''}
                    </div>`).join('') : '<p class="text-gray-400 text-sm text-center py-2">لا توجد زيارات متابعة بعد</p>'}
                </div>
                <button id="record-delivery-btn" class="w-full mt-3 text-sm text-gray-500 hover:text-gray-700 underline">تسجيل الولادة (إنهاء متابعة الحمل)</button>
                <div id="delivery-form-container"></div>
            `;
            document.getElementById('add-anc-btn').addEventListener('click', () => this.showAncVisitForm(pregnancy.id));
            document.getElementById('record-delivery-btn').addEventListener('click', () => this.showDeliveryForm(pregnancy.id));
        } catch (e) {
            el.innerHTML = `<p class="text-red-500 text-sm">تعذّر تحميل بيانات الحمل: ${esc(e.message)}</p>`;
        }
    }

    showNewPregnancyForm() {
        const container = document.getElementById('pregnancy-form-container');
        container.innerHTML = `
            <form id="new-pregnancy-form" class="space-y-2 mt-3 bg-gray-50 p-3 rounded-lg">
                <label class="text-xs text-gray-600">تاريخ آخر دورة شهرية (LMP)</label>
                <input name="lmp_date" type="date" required class="w-full px-3 py-2 border rounded-lg text-sm">
                <div class="grid grid-cols-3 gap-2">
                    <input name="gravida" type="number" placeholder="G (مرات الحمل)" class="px-3 py-2 border rounded-lg text-sm">
                    <input name="para" type="number" placeholder="P (الولادات)" class="px-3 py-2 border rounded-lg text-sm">
                    <input name="abortions" type="number" placeholder="A (الإجهاض)" class="px-3 py-2 border rounded-lg text-sm">
                </div>
                <label class="flex items-center gap-2 text-sm"><input type="checkbox" name="is_multiple"> حمل متعدد (توأم)</label>
                <textarea name="risk_factors" placeholder="عوامل الخطورة (اختياري)" class="w-full px-3 py-2 border rounded-lg text-sm"></textarea>
                <button type="submit" class="w-full bg-pink-500 text-white py-2 rounded-lg text-sm hover:bg-pink-600">حفظ</button>
            </form>`;
        document.getElementById('new-pregnancy-form').addEventListener('submit', (e) => this.createPregnancy(e));
    }

    async createPregnancy(event) {
        event.preventDefault();
        const form = new FormData(event.target);
        try {
            const payload = {
                patient_id: this.selectedPatient.id,
                clinic_id: this.profile.clinic_id,
                lmp_date: form.get('lmp_date'),
                gravida: form.get('gravida') ? parseInt(form.get('gravida')) : null,
                para: form.get('para') ? parseInt(form.get('para')) : null,
                abortions: form.get('abortions') ? parseInt(form.get('abortions')) : null,
                is_multiple: form.get('is_multiple') === 'on',
                risk_factors: form.get('risk_factors') || null,
            };
            await pgFetch('pregnancies', { method: 'POST', body: payload, accessToken: this.token });
            showMessage('تم بدء متابعة الحمل بنجاح', 'success');
            this.loadPatientPregnancy(this.selectedPatient.id, this.token);
        } catch (e) { showMessage('خطأ: ' + e.message, 'error'); }
    }

    showAncVisitForm(pregnancyId) {
        const container = document.getElementById('anc-form-container');
        container.innerHTML = `
            <form id="anc-visit-form" class="space-y-2 mb-4 bg-gray-50 p-3 rounded-lg">
                <div class="grid grid-cols-2 gap-2">
                    <input name="weight" type="number" step="0.1" placeholder="الوزن (كجم)" class="px-3 py-2 border rounded-lg text-sm">
                    <input name="fetal_heart_rate" type="number" placeholder="نبض الجنين" class="px-3 py-2 border rounded-lg text-sm">
                </div>
                <div class="grid grid-cols-2 gap-2">
                    <input name="bp_systolic" type="number" placeholder="الضغط الانقباضي" class="px-3 py-2 border rounded-lg text-sm">
                    <input name="bp_diastolic" type="number" placeholder="الضغط الانبساطي" class="px-3 py-2 border rounded-lg text-sm">
                </div>
                <input name="fundal_height_cm" type="number" step="0.1" placeholder="ارتفاع قاع الرحم (سم)" class="w-full px-3 py-2 border rounded-lg text-sm">
                <textarea name="notes" placeholder="ملاحظات" class="w-full px-3 py-2 border rounded-lg text-sm"></textarea>
                <button type="submit" class="w-full bg-pink-500 text-white py-2 rounded-lg text-sm hover:bg-pink-600">حفظ الزيارة</button>
            </form>`;
        document.getElementById('anc-visit-form').addEventListener('submit', (e) => this.createAncVisit(e, pregnancyId));
    }

    async createAncVisit(event, pregnancyId) {
        event.preventDefault();
        const form = new FormData(event.target);
        try {
            const payload = {
                pregnancy_id: pregnancyId,
                doctor_id: this.profile.id,
                weight: form.get('weight') ? parseFloat(form.get('weight')) : null,
                bp_systolic: form.get('bp_systolic') ? parseFloat(form.get('bp_systolic')) : null,
                bp_diastolic: form.get('bp_diastolic') ? parseFloat(form.get('bp_diastolic')) : null,
                fetal_heart_rate: form.get('fetal_heart_rate') ? parseInt(form.get('fetal_heart_rate')) : null,
                fundal_height_cm: form.get('fundal_height_cm') ? parseFloat(form.get('fundal_height_cm')) : null,
                notes: form.get('notes') || null,
            };
            await pgFetch('anc_visits', { method: 'POST', body: payload, accessToken: this.token });
            showMessage('تم تسجيل زيارة المتابعة', 'success');
            this.loadPatientPregnancy(this.selectedPatient.id, this.token);
        } catch (e) { showMessage('خطأ: ' + e.message, 'error'); }
    }

    showDeliveryForm(pregnancyId) {
        const container = document.getElementById('delivery-form-container');
        container.innerHTML = `
            <form id="delivery-form" class="space-y-2 mt-3 bg-gray-50 p-3 rounded-lg">
                <input name="delivery_date" type="date" required class="w-full px-3 py-2 border rounded-lg text-sm">
                <select name="delivery_type" class="w-full px-3 py-2 border rounded-lg text-sm">
                    <option value="vaginal">ولادة طبيعية</option>
                    <option value="c_section">قيصرية</option>
                    <option value="assisted">ولادة بمساعدة</option>
                </select>
                <input name="hospital" placeholder="المستشفى" class="w-full px-3 py-2 border rounded-lg text-sm">
                <div class="grid grid-cols-2 gap-2">
                    <select name="baby_gender" class="px-3 py-2 border rounded-lg text-sm">
                        <option value="male">ذكر</option>
                        <option value="female">أنثى</option>
                    </select>
                    <input name="baby_weight_grams" type="number" placeholder="وزن المولود (جم)" class="px-3 py-2 border rounded-lg text-sm">
                </div>
                <button type="submit" class="w-full bg-green-600 text-white py-2 rounded-lg text-sm hover:bg-green-700">تأكيد وإنهاء متابعة الحمل</button>
            </form>`;
        document.getElementById('delivery-form').addEventListener('submit', (e) => this.recordDelivery(e, pregnancyId));
    }

    async recordDelivery(event, pregnancyId) {
        event.preventDefault();
        const form = new FormData(event.target);
        try {
            const payload = {
                pregnancy_id: pregnancyId,
                delivery_date: form.get('delivery_date'),
                delivery_type: form.get('delivery_type'),
                hospital: form.get('hospital') || null,
                baby_gender: form.get('baby_gender'),
                baby_weight_grams: form.get('baby_weight_grams') ? parseFloat(form.get('baby_weight_grams')) : null,
            };
            await pgFetch('deliveries', { method: 'POST', body: payload, accessToken: this.token });
            showMessage('تم تسجيل الولادة بنجاح', 'success');
            this.loadPatientPregnancy(this.selectedPatient.id, this.token);
        } catch (e) { showMessage('خطأ: ' + e.message, 'error'); }
    }

    // ---------- Chronic conditions module (يشمل السرطان وغيره) ----------
    async loadPatientChronic(id, token) {
        const el = document.getElementById('chronic-section');
        if (!el) return;
        try {
            const conditions = await pgFetch(`chronic_conditions?patient_id=eq.${id}&select=*&order=created_at.desc`, { accessToken: token });
            const optionsHtml = COMMON_CONDITIONS.map(c => `<option value="${esc(c)}">`).join('');
            el.innerHTML = `
                <button id="add-condition-btn" class="w-full bg-amber-500 text-white py-2 rounded-lg text-sm hover:bg-amber-600 mb-3">+ تسجيل مرض مزمن (سكري، ضغط، سرطان، ...)</button>
                <div id="condition-form-container"></div>
                <datalist id="condition-suggestions">${optionsHtml}</datalist>
                <div id="conditions-list">
                    ${(conditions && conditions.length) ? conditions.map(c => `
                        <div class="border border-gray-100 rounded-lg p-3 mb-2">
                            <div class="flex justify-between items-center">
                                <span class="font-semibold text-sm">${esc(c.condition_name)}</span>
                                <span class="text-xs px-2 py-1 rounded-full ${c.status === 'active' ? 'bg-red-50 text-red-600' : c.status === 'controlled' ? 'bg-green-50 text-green-600' : 'bg-gray-100 text-gray-500'}">${esc(c.status)}</span>
                            </div>
                            ${c.diagnosed_date ? `<p class="text-xs text-gray-400 mt-1">تاريخ التشخيص: ${fmtDate(c.diagnosed_date)}</p>` : ''}
                            <button data-cid="${c.id}" class="record-reading-btn text-xs text-blue-600 hover:underline mt-2">+ تسجيل قياسة مرتبطة</button>
                            <div class="reading-form-slot" data-slot="${c.id}"></div>
                        </div>`).join('') : '<p class="text-gray-400 text-sm text-center py-4">لا توجد حالات مزمنة مسجلة</p>'}
                </div>
                <h4 class="text-sm font-bold text-gray-700 mt-4 mb-2">سجل جميع القياسات (سكر / ضغط / أخرى)</h4>
                <div id="all-readings-list"></div>
            `;
            document.getElementById('add-condition-btn').addEventListener('click', () => this.showNewConditionForm());
            el.querySelectorAll('.record-reading-btn').forEach(btn => {
                btn.addEventListener('click', () => this.showReadingForm(btn.dataset.cid));
            });
            this.loadAllReadings(id, token);
        } catch (e) {
            el.innerHTML = `<p class="text-red-500 text-sm">تعذّر تحميل الأمراض المزمنة: ${esc(e.message)}</p>`;
        }
    }

    showNewConditionForm() {
        const container = document.getElementById('condition-form-container');
        container.innerHTML = `
            <form id="new-condition-form" class="space-y-2 mb-4 bg-gray-50 p-3 rounded-lg">
                <input name="condition_name" list="condition-suggestions" placeholder="اسم المرض (اختر من القائمة أو اكتب اسمًا آخر)" required class="w-full px-3 py-2 border rounded-lg text-sm">
                <input name="diagnosed_date" type="date" class="w-full px-3 py-2 border rounded-lg text-sm">
                <select name="status" class="w-full px-3 py-2 border rounded-lg text-sm">
                    <option value="active">نشط</option>
                    <option value="controlled">تحت السيطرة</option>
                    <option value="resolved">تم الشفاء</option>
                </select>
                <textarea name="notes" placeholder="ملاحظات (نوع العلاج، المرحلة، إلخ)" class="w-full px-3 py-2 border rounded-lg text-sm"></textarea>
                <button type="submit" class="w-full bg-amber-500 text-white py-2 rounded-lg text-sm hover:bg-amber-600">حفظ</button>
            </form>`;
        document.getElementById('new-condition-form').addEventListener('submit', (e) => this.createCondition(e));
    }

    async createCondition(event) {
        event.preventDefault();
        const form = new FormData(event.target);
        try {
            const payload = {
                patient_id: this.selectedPatient.id,
                clinic_id: this.profile.clinic_id,
                condition_name: form.get('condition_name'),
                diagnosed_date: form.get('diagnosed_date') || null,
                status: form.get('status'),
                notes: form.get('notes') || null,
            };
            await pgFetch('chronic_conditions', { method: 'POST', body: payload, accessToken: this.token });
            showMessage('تم تسجيل الحالة المزمنة', 'success');
            this.loadPatientChronic(this.selectedPatient.id, this.token);
        } catch (e) { showMessage('خطأ: ' + e.message, 'error'); }
    }

    showReadingForm(conditionId) {
        const slot = document.querySelector(`.reading-form-slot[data-slot="${conditionId}"]`);
        if (!slot) return;
        slot.innerHTML = `
            <form class="reading-form space-y-2 mt-2 bg-white border rounded-lg p-2">
                <select name="reading_type" class="w-full px-2 py-1.5 border rounded text-xs">
                    <option value="blood_pressure">ضغط الدم</option>
                    <option value="blood_glucose">سكر الدم</option>
                    <option value="weight">الوزن</option>
                    <option value="hba1c">HbA1c</option>
                    <option value="other">قيمة أخرى</option>
                </select>
                <div class="grid grid-cols-2 gap-1">
                    <input name="systolic" type="number" placeholder="الانقباضي" class="px-2 py-1.5 border rounded text-xs">
                    <input name="diastolic" type="number" placeholder="الانبساطي" class="px-2 py-1.5 border rounded text-xs">
                </div>
                <input name="glucose_value" type="number" placeholder="قيمة السكر" class="w-full px-2 py-1.5 border rounded text-xs">
                <select name="glucose_context" class="w-full px-2 py-1.5 border rounded text-xs">
                    <option value="fasting">صائم</option>
                    <option value="postprandial">بعد الأكل</option>
                    <option value="random">عشوائي</option>
                </select>
                <input name="value" type="number" placeholder="القيمة (عام)" class="w-full px-2 py-1.5 border rounded text-xs">
                <input name="unit" placeholder="الوحدة" class="w-full px-2 py-1.5 border rounded text-xs">
                <button type="submit" class="w-full bg-blue-600 text-white py-1.5 rounded text-xs hover:bg-blue-700">حفظ القياسة</button>
            </form>`;
        slot.querySelector('.reading-form').addEventListener('submit', (e) => this.createReading(e, conditionId));
    }

    async createReading(event, conditionId) {
        event.preventDefault();
        const form = new FormData(event.target);
        const type = form.get('reading_type');
        const payload = {
            clinic_id: this.profile.clinic_id,
            patient_id: this.selectedPatient.id,
            chronic_condition_id: conditionId || null,
            reading_type: type,
            recorded_by: this.profile.id,
        };
        if (type === 'blood_pressure') {
            payload.systolic = parseFloat(form.get('systolic'));
            payload.diastolic = parseFloat(form.get('diastolic'));
        } else if (type === 'blood_glucose') {
            payload.glucose_value = parseFloat(form.get('glucose_value'));
            payload.glucose_context = form.get('glucose_context');
        } else {
            payload.value = form.get('value') ? parseFloat(form.get('value')) : null;
            payload.unit = form.get('unit') || null;
        }
        try {
            await pgFetch('vital_readings', { method: 'POST', body: payload, accessToken: this.token });
            showMessage('تم تسجيل القياسة', 'success');
            this.loadPatientChronic(this.selectedPatient.id, this.token);
        } catch (e) { showMessage('خطأ: ' + e.message, 'error'); }
    }

    async loadAllReadings(id, token) {
        try {
            const readings = await pgFetch(`vital_readings?patient_id=eq.${id}&select=*&order=reading_date.desc&limit=30`, { accessToken: token });
            document.getElementById('all-readings-list').innerHTML = (readings && readings.length)
                ? readings.map(r => `
                    <div class="border-b border-gray-100 py-2 flex justify-between text-sm">
                        <span>
                            ${r.reading_type === 'blood_pressure' ? `ضغط: ${r.systolic}/${r.diastolic}` : ''}
                            ${r.reading_type === 'blood_glucose' ? `سكر: ${r.glucose_value} (${r.glucose_context || ''})` : ''}
                            ${!['blood_pressure', 'blood_glucose'].includes(r.reading_type) ? `${r.reading_type}: ${r.value ?? ''} ${r.unit ?? ''}` : ''}
                        </span>
                        <span class="text-xs ${r.is_abnormal === 'high' || r.is_abnormal === 'critical' ? 'text-red-600 font-semibold' : 'text-gray-400'}">
                            ${fmtDate(r.reading_date)} ${r.is_abnormal ? '· ' + r.is_abnormal : ''}
                        </span>
                    </div>`).join('')
                : '<p class="text-gray-400 text-sm text-center py-4">لا توجد قياسات مسجلة</p>';
        } catch (e) { console.error(e); }
    }

    // ---------- Appointments ----------
    async loadAppointments() {
        try {
            const data = await pgFetch('appointments?select=*,patients(name)&order=start_time.asc', { accessToken: this.token });
            const container = document.getElementById('appointments-list');
            container.innerHTML = (data && data.length)
                ? data.map(a => `
                    <div class="bg-white p-4 rounded-xl shadow-sm border border-gray-100 mb-3">
                        <div class="flex justify-between items-center">
                            <div>
                                <h3 class="font-bold text-gray-800">${esc(a.patients?.name || 'مريض')}</h3>
                                <p class="text-sm text-gray-500">${new Date(a.start_time).toLocaleString('ar-EG')}</p>
                            </div>
                            <span class="bg-green-50 text-green-700 px-3 py-1 rounded-full text-xs">${esc(a.status)}</span>
                        </div>
                    </div>`).join('')
                : '<p class="text-gray-400 text-center py-10">لا توجد مواعيد بعد</p>';
        } catch (e) { showMessage('تعذّر تحميل المواعيد: ' + e.message, 'error'); }

        try {
            const patients = await pgFetch('patients?select=id,name,patient_number', { accessToken: this.token });
            this._patientsCache = patients || [];
            const list = document.getElementById('appointment-patient-list');
            if (list) list.innerHTML = this._patientsCache.map(p => `<option value="${esc(p.name)} (${esc(p.patient_number)})" data-id="${p.id}">`).join('');
        } catch (e) { console.error(e); }
    }

    async createAppointment(event) {
        event.preventDefault();
        const form = new FormData(event.target);
        const patientLabel = form.get('patient_search');
        const match = this._patientsCache.find(p => `${p.name} (${p.patient_number})` === patientLabel);
        if (!match) { showMessage('اختر مريضًا من القائمة المقترحة', 'error'); return; }
        try {
            const branches = await pgFetch(`branches?clinic_id=eq.${this.profile.clinic_id}&select=id&limit=1`, { accessToken: this.token });
            const branchId = branches && branches[0] ? branches[0].id : null;
            const payload = {
                clinic_id: this.profile.clinic_id,
                branch_id: branchId,
                patient_id: match.id,
                doctor_id: this.profile.id,
                start_time: form.get('start_time'),
                type: form.get('type') || 'followup',
                notes: form.get('notes') || null,
            };
            await pgFetch('appointments', { method: 'POST', body: payload, accessToken: this.token });
            showMessage('تم حجز الموعد بنجاح', 'success');
            event.target.reset();
            this.loadAppointments();
        } catch (e) { showMessage('خطأ: ' + e.message, 'error'); }
    }

    // ---------- Render shell ----------
    render() {
        const app = document.getElementById('app');
        if (!this.profile) {
            app.innerHTML = this.renderAuthPages();
            this.bindAuthForms();
            return;
        }
        const theme = this.specialtyTheme;
        app.innerHTML = `
            <nav class="bg-white shadow-sm sticky top-0 z-40 border-b-4 border-${theme.color}-500">
                <div class="max-w-5xl mx-auto px-4">
                    <div class="flex justify-between h-16 items-center">
                        <h1 class="text-lg font-bold text-${theme.color}-600 flex items-center gap-2">${theme.icon} ${esc(this.clinic?.name || 'ClinicFlow')}</h1>
                        <div class="flex items-center gap-1 overflow-x-auto">
                            <button data-nav="dashboard" class="nav-btn px-3 py-2 rounded-lg text-sm ${this.currentPage === 'dashboard' ? `bg-${theme.color}-600 text-white` : 'text-gray-600 hover:bg-gray-100'}">الرئيسية</button>
                            <button data-nav="patients" class="nav-btn px-3 py-2 rounded-lg text-sm ${['patients', 'patient-detail'].includes(this.currentPage) ? `bg-${theme.color}-600 text-white` : 'text-gray-600 hover:bg-gray-100'}">المرضى</button>
                            <button data-nav="appointments" class="nav-btn px-3 py-2 rounded-lg text-sm ${this.currentPage === 'appointments' ? `bg-${theme.color}-600 text-white` : 'text-gray-600 hover:bg-gray-100'}">المواعيد</button>
                            <button id="logout-btn" class="px-3 py-2 rounded-lg text-sm text-red-600 hover:bg-red-50">خروج</button>
                        </div>
                    </div>
                </div>
            </nav>
            <main class="max-w-5xl mx-auto px-4 py-6">${this.renderCurrentPage()}</main>
        `;
        app.querySelectorAll('.nav-btn').forEach(btn => btn.addEventListener('click', () => this.navigate(btn.dataset.nav)));
        document.getElementById('logout-btn').addEventListener('click', () => this.logout());
        this.bindPageForms();
    }

    renderAuthPages() {
        if (this.currentPage === 'complete-signup') {
            return `
            <div class="min-h-screen flex items-center justify-center bg-gray-50 p-4">
                <div class="bg-white p-8 rounded-2xl shadow-lg w-full max-w-md">
                    <h1 class="text-2xl font-bold text-blue-600 mb-2">🏥 استكمال إنشاء العيادة</h1>
                    <p class="text-sm text-gray-500 mb-4">حسابك موجود لكن لم يتم العثور على ملف العيادة. أكمل البيانات:</p>
                    ${this._lastProfileError ? `<div class="bg-red-50 border border-red-200 text-red-700 text-sm p-3 rounded-lg mb-4">تفاصيل تقنية: ${esc(this._lastProfileError)}</div>` : ''}
                    <form id="complete-signup-form" class="space-y-3">
                        <input name="clinic_name" placeholder="اسم العيادة" required class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                        <input name="admin_full_name" placeholder="اسم المدير/الطبيب" required class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                        <select name="specialty" required class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                            <option value="">اختر تخصص العيادة</option>
                            <option value="obgyn">نساء وتوليد (حمل وولادة)</option>
                            <option value="chronic">متابعة أمراض مزمنة (سكري، ضغط، ...)</option>
                            <option value="oncology">أورام (سرطان)</option>
                            <option value="general">عيادة عامة (كل الوحدات)</option>
                        </select>
                        <button type="submit" class="w-full bg-blue-600 text-white py-2.5 rounded-lg hover:bg-blue-700">إنشاء العيادة</button>
                    </form>
                    <button id="complete-signup-logout" class="w-full mt-4 text-red-500 text-sm hover:underline">تسجيل الخروج والعودة لصفحة الدخول</button>
                </div>
            </div>`;
        }
        if (this.currentPage === 'register') {
            return `
            <div class="min-h-screen flex items-center justify-center bg-gray-50 p-4">
                <div class="bg-white p-8 rounded-2xl shadow-lg w-full max-w-md">
                    <h1 class="text-2xl font-bold text-blue-600 mb-6">🏥 إنشاء عيادة جديدة</h1>
                    <form id="register-form" class="space-y-3">
                        <input name="clinic_name" placeholder="اسم العيادة" required class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                        <input name="admin_full_name" placeholder="اسم المدير/الطبيب" required class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                        <select name="specialty" required class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                            <option value="">اختر تخصص العيادة</option>
                            <option value="obgyn">نساء وتوليد (حمل وولادة)</option>
                            <option value="chronic">متابعة أمراض مزمنة (سكري، ضغط، ...)</option>
                            <option value="oncology">أورام (سرطان)</option>
                            <option value="general">عيادة عامة (كل الوحدات)</option>
                        </select>
                        <input name="email" type="email" placeholder="البريد الإلكتروني" required class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                        <input name="password" type="password" placeholder="كلمة المرور (6 أحرف على الأقل)" required minlength="6" class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                        <button type="submit" class="w-full bg-blue-600 text-white py-2.5 rounded-lg hover:bg-blue-700">إنشاء العيادة</button>
                    </form>
                    <p class="mt-4 text-sm text-center"><a href="#" id="go-login" class="text-blue-600">لديك حساب بالفعل؟ سجّل الدخول</a></p>
                </div>
            </div>`;
        }
        return `
        <div class="min-h-screen flex items-center justify-center bg-gray-50 p-4">
            <div class="bg-white p-8 rounded-2xl shadow-lg w-full max-w-md">
                <h1 class="text-2xl font-bold text-blue-600 mb-6">🏥 ClinicFlow</h1>
                <form id="login-form" class="space-y-3">
                    <input name="email" type="email" placeholder="البريد الإلكتروني" required class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                    <input name="password" type="password" placeholder="كلمة المرور" required class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                    <button type="submit" class="w-full bg-blue-600 text-white py-2.5 rounded-lg hover:bg-blue-700">دخول</button>
                </form>
                <p class="mt-4 text-sm text-center"><a href="#" id="go-register" class="text-blue-600">عيادة جديدة؟ أنشئ حسابًا</a></p>
            </div>
        </div>`;
    }

    bindAuthForms() {
        const loginForm = document.getElementById('login-form');
        if (loginForm) loginForm.addEventListener('submit', (e) => this.login(e));
        const registerForm = document.getElementById('register-form');
        if (registerForm) registerForm.addEventListener('submit', (e) => this.registerAccount(e));
        const completeForm = document.getElementById('complete-signup-form');
        if (completeForm) completeForm.addEventListener('submit', (e) => this.completeSignup(e));
        const completeLogout = document.getElementById('complete-signup-logout');
        if (completeLogout) completeLogout.addEventListener('click', () => this.logout());
        const goRegister = document.getElementById('go-register');
        if (goRegister) goRegister.addEventListener('click', (e) => { e.preventDefault(); this.currentPage = 'register'; this.render(); });
        const goLogin = document.getElementById('go-login');
        if (goLogin) goLogin.addEventListener('click', (e) => { e.preventDefault(); this.currentPage = 'login'; this.render(); });
    }

    bindPageForms() {
        const patientForm = document.getElementById('patient-form');
        if (patientForm) patientForm.addEventListener('submit', (e) => this.createPatient(e));
        const backBtn = document.getElementById('back-to-patients');
        if (backBtn) backBtn.addEventListener('click', () => this.navigate('patients'));
        const apptForm = document.getElementById('appointment-form');
        if (apptForm) apptForm.addEventListener('submit', (e) => this.createAppointment(e));
    }

    renderCurrentPage() {
        switch (this.currentPage) {
            case 'patients': return this.renderPatientsPage();
            case 'appointments': return this.renderAppointmentsPage();
            case 'patient-detail': return this.renderPatientDetailPage();
            default: return this.renderDashboard();
        }
    }

    renderDashboard() {
        const specialty = this.clinic?.specialty || 'general';
        const showPregnancy = specialty === 'obgyn' || specialty === 'general';
        const showChronic = specialty === 'chronic' || specialty === 'general';
        const showOncology = specialty === 'oncology' || specialty === 'general';
        const theme = this.specialtyTheme;

        const statCards = [
            `<div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                <h3 class="text-gray-400 text-xs mb-1">مواعيد اليوم</h3>
                <p id="today-appointments" class="text-2xl font-bold text-${theme.color}-600">0</p>
            </div>`,
            `<div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                <h3 class="text-gray-400 text-xs mb-1">إجمالي المرضى</h3>
                <p id="total-patients" class="text-2xl font-bold text-green-600">0</p>
            </div>`,
            `<div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                <h3 class="text-gray-400 text-xs mb-1">إجمالي الزيارات</h3>
                <p id="total-visits" class="text-2xl font-bold text-purple-600">0</p>
            </div>`,
        ];
        if (showPregnancy) {
            statCards.push(`<div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                <h3 class="text-gray-400 text-xs mb-1">حالات حمل نشطة</h3>
                <p id="active-pregnancies" class="text-2xl font-bold text-pink-600">0</p>
            </div>`);
        }

        const sections = [];
        if (showPregnancy) {
            sections.push(`<div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                <h2 class="font-bold text-gray-800 mb-3">🤰 متابعة الحمل النشطة</h2>
                <div id="active-pregnancies-list"></div>
            </div>`);
        }
        if (showChronic) {
            sections.push(`<div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                <h2 class="font-bold text-gray-800 mb-3">💊 متابعة الأمراض المزمنة (سكري، ضغط...)</h2>
                <div id="chronic-conditions-list"></div>
            </div>`);
        }
        if (showOncology) {
            sections.push(`<div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                <h2 class="font-bold text-gray-800 mb-3">🎗️ متابعة حالات الأورام</h2>
                <div id="oncology-conditions-list"></div>
            </div>`);
        }

        // إخفاء العنصر الشرطي عند عدم توفره حتى لا يفشل loadDashboard في تحديثه
        if (!showPregnancy) {
            sections.push('<div id="active-pregnancies-list" style="display:none"></div>');
        }
        if (!showChronic) {
            sections.push('<div id="chronic-conditions-list" style="display:none"></div>');
        }
        if (!showOncology) {
            sections.push('<div id="oncology-conditions-list" style="display:none"></div>');
        }
        if (!showPregnancy) {
            statCards.push('<p id="active-pregnancies" style="display:none"></p>');
        }

        return `
            <div class="grid grid-cols-2 md:grid-cols-4 gap-4 mb-6">${statCards.join('')}</div>
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4">${sections.join('')}</div>
        `;
    }

    renderPatientsPage() {
        return `
            <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
                <div class="lg:col-span-1">
                    <div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                        <h2 class="font-bold text-gray-800 mb-4">إضافة مريض جديد</h2>
                        <form id="patient-form" class="space-y-3">
                            <input type="text" name="name" placeholder="اسم المريض" required class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                            <select name="gender" class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                                <option value="female">أنثى</option>
                                <option value="male">ذكر</option>
                            </select>
                            <input type="text" name="phone" placeholder="رقم الهاتف" required class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                            <input type="date" name="date_of_birth" class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                            <input type="text" name="allergies" placeholder="الحساسية (إن وجدت)" class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                            <input type="text" name="chronic_conditions" placeholder="ملاحظات عامة (اختياري)" class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                            <button type="submit" class="w-full bg-blue-600 text-white py-2.5 rounded-lg hover:bg-blue-700">إضافة</button>
                        </form>
                    </div>
                </div>
                <div class="lg:col-span-2">
                    <h2 class="font-bold text-gray-800 mb-4">قائمة المرضى</h2>
                    <div id="patients-list"></div>
                </div>
            </div>
        `;
    }

    renderAppointmentsPage() {
        return `
            <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
                <div class="lg:col-span-1">
                    <div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                        <h2 class="font-bold text-gray-800 mb-4">حجز موعد جديد</h2>
                        <form id="appointment-form" class="space-y-3">
                            <input name="patient_search" list="appointment-patient-list" placeholder="ابحث عن اسم المريض" required class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                            <datalist id="appointment-patient-list"></datalist>
                            <input name="start_time" type="datetime-local" required class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                            <select name="type" class="w-full px-4 py-2.5 border border-gray-200 rounded-lg">
                                <option value="new">كشف جديد</option>
                                <option value="followup">متابعة</option>
                                <option value="anc">متابعة حمل</option>
                                <option value="chronic_followup">متابعة مرض مزمن</option>
                            </select>
                            <textarea name="notes" placeholder="ملاحظات" class="w-full px-4 py-2.5 border border-gray-200 rounded-lg"></textarea>
                            <button type="submit" class="w-full bg-blue-600 text-white py-2.5 rounded-lg hover:bg-blue-700">حجز</button>
                        </form>
                    </div>
                </div>
                <div class="lg:col-span-2">
                    <h2 class="font-bold text-gray-800 mb-4">المواعيد</h2>
                    <div id="appointments-list"></div>
                </div>
            </div>
        `;
    }

    renderPatientDetailPage() {
        const specialty = this.clinic?.specialty || 'general';
        const showPregnancy = specialty === 'obgyn' || specialty === 'general';
        const showChronic = specialty === 'chronic' || specialty === 'oncology' || specialty === 'general';
        const chronicTitle = specialty === 'oncology' ? 'متابعة الأورام والقياسات' : 'الأمراض المزمنة والقياسات';

        const columns = [`
            <div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                <h3 class="font-bold text-gray-800 mb-3">سجل الزيارات</h3>
                <div id="patient-visits"></div>
            </div>`];
        if (showPregnancy) {
            columns.push(`
            <div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                <h3 class="font-bold text-gray-800 mb-3">🤰 متابعة الحمل</h3>
                <div id="pregnancy-section"></div>
            </div>`);
        } else {
            columns.push('<div id="pregnancy-section" style="display:none"></div>');
        }
        if (showChronic) {
            columns.push(`
            <div class="bg-white p-5 rounded-xl shadow-sm border border-gray-100">
                <h3 class="font-bold text-gray-800 mb-3">💊 ${chronicTitle}</h3>
                <div id="chronic-section"></div>
            </div>`);
        } else {
            columns.push('<div id="chronic-section" style="display:none"></div>');
        }

        return `
            <button id="back-to-patients" class="mb-4 text-blue-600 text-sm">&rarr; رجوع لقائمة المرضى</button>
            <div id="patient-detail-header" class="bg-white p-5 rounded-xl shadow-sm border border-gray-100 mb-6"></div>
            <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">${columns.join('')}</div>
        `;
    }
}

const app = new ClinicApp();
